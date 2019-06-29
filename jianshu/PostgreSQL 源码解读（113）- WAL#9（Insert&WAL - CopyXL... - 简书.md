本节重点跟踪分析了ReserveXLogInsertLocation和CopyXLogRecordToWAL函数的实现逻辑,ReserveXLogInsertLocation函数为XLOG
Record预留合适的空间,CopyXLogRecordToWAL则负责拷贝XLOG Record到WAL buffer的保留空间中。

### 一、数据结构

**全局变量**

    
    
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
    
    /*
     * Private, possibly out-of-date copy of shared LogwrtResult.
     * See discussion above.
     * 进程私有的可能已过期的共享LogwrtResult变量的拷贝.
     */
    static XLogwrtResult LogwrtResult = {0, 0};
    
    /* The number of bytes in a WAL segment usable for WAL data. */
    //WAL segment file中可用于WAL data的字节数(不包括page header)
    static int UsableBytesInSegment;
    
    

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
    
    
    #define XLogSegmentOffset(xlogptr, wal_segsz_bytes) \
        ((xlogptr) & ((wal_segsz_bytes) - 1))
    /*
     * Calculate the amount of space left on the page after 'endptr'. Beware
     * multiple evaluation!
     * 计算page中在"endptr"后的剩余空闲空间.注意multiple evaluation! 
     */
    #define INSERT_FREESPACE(endptr)    \
        (((endptr) % XLOG_BLCKSZ == 0) ? 0 : (XLOG_BLCKSZ - (endptr) % XLOG_BLCKSZ))
    
    

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
    
    

### 二、源码解读

**ReserveXLogInsertLocation**  
在WAL(buffer)中为给定大小的记录预留合适的空间。*StartPos设置为预留部分的开头，*EndPos设置为其结尾+1。*PrePtr设置为前一记录的开头;它用于设置该记录的xl_prev变量。

    
    
    /*
     * Reserves the right amount of space for a record of given size from the WAL.
     * *StartPos is set to the beginning of the reserved section, *EndPos to
     * its end+1. *PrevPtr is set to the beginning of the previous record; it is
     * used to set the xl_prev of this record.
     * 在WAL(buffer)中为给定大小的记录预留合适的空间。
     * *StartPos设置为预留部分的开头，*EndPos设置为其结尾+1。
     * *PrePtr设置为前一记录的开头;它用于设置该记录的xl_prev。
     *
     * This is the performance critical part of XLogInsert that must be serialized
     * across backends. The rest can happen mostly in parallel. Try to keep this
     * section as short as possible, insertpos_lck can be heavily contended on a
     * busy system.
     * 这是XLogInsert中与性能密切相关的部分，必须在后台进程之间序列执行。
     * 其余的大部分可以同时发生。
     * 尽量精简这部分的逻辑，insertpos_lck可以在繁忙的系统上存在激烈的竞争。
     *
     * NB: The space calculation here must match the code in CopyXLogRecordToWAL,
     * where we actually copy the record to the reserved space.
     * 注意:这里计算的空间必须与CopyXLogRecordToWAL()函数一致,
     *   在CopyXLogRecordToWAL中会实际拷贝数据到预留空间中.
     */
    static void
    ReserveXLogInsertLocation(int size, XLogRecPtr *StartPos, XLogRecPtr *EndPos,
                              XLogRecPtr *PrevPtr)
    {
        XLogCtlInsert *Insert = &XLogCtl->Insert;//插入控制器
        uint64      startbytepos;//开始位置
        uint64      endbytepos;//结束位置
        uint64      prevbytepos;//上一位置
    
        size = MAXALIGN(size);//大小对齐
    
        /* All (non xlog-switch) records should contain data. */
        //除了xlog-switch外,所有的记录都应该包含数据.
        Assert(size > SizeOfXLogRecord);
    
        /*
         * The duration the spinlock needs to be held is minimized by minimizing
         * the calculations that have to be done while holding the lock. The
         * current tip of reserved WAL is kept in CurrBytePos, as a byte position
         * that only counts "usable" bytes in WAL, that is, it excludes all WAL
         * page headers. The mapping between "usable" byte positions and physical
         * positions (XLogRecPtrs) can be done outside the locked region, and
         * because the usable byte position doesn't include any headers, reserving
         * X bytes from WAL is almost as simple as "CurrBytePos += X".
         * spinlock需要持有的时间通过最小化必须持有锁的计算逻辑达到最小化。
         * 预留的WAL空间通过CurrBytePos变量(大小一个字节)保存，
         *   它只计算WAL中的“可用”字节，也就是说，它排除了所有的WAL page header。
         * “可用”字节位置和物理位置(XLogRecPtrs)之间的映射可以在锁定区域之外完成，
         *   而且由于可用字节位置不包含任何header，从WAL预留X字节的大小几乎和“CurrBytePos += X”一样简单。
         */
        SpinLockAcquire(&Insert->insertpos_lck);//申请锁
        //开始位置
        startbytepos = Insert->CurrBytePos;
        //结束位置
        endbytepos = startbytepos + size;
        //上一位置
        prevbytepos = Insert->PrevBytePos;
        //调整控制器的相关变量
        Insert->CurrBytePos = endbytepos;
        Insert->PrevBytePos = startbytepos;
        //释放锁
        SpinLockRelease(&Insert->insertpos_lck);
        //返回值
        //计算开始/结束/上一位置偏移
        *StartPos = XLogBytePosToRecPtr(startbytepos);
        *EndPos = XLogBytePosToEndRecPtr(endbytepos);
        *PrevPtr = XLogBytePosToRecPtr(prevbytepos);
    
        /*
         * Check that the conversions between "usable byte positions" and
         * XLogRecPtrs work consistently in both directions.
         * 检查双向转换之后的值是一致的.
         */
        Assert(XLogRecPtrToBytePos(*StartPos) == startbytepos);
        Assert(XLogRecPtrToBytePos(*EndPos) == endbytepos);
        Assert(XLogRecPtrToBytePos(*PrevPtr) == prevbytepos);
    }
     
    /*
     * Converts a "usable byte position" to XLogRecPtr. A usable byte position
     * is the position starting from the beginning of WAL, excluding all WAL
     * page headers.
     * 将“可用字节位置”转换为XLogRecPtr。
     * 可用字节位置是从WAL开始的位置，不包括所有WAL page header。
     */
    static XLogRecPtr
    XLogBytePosToRecPtr(uint64 bytepos)
    {
        uint64      fullsegs;
        uint64      fullpages;
        uint64      bytesleft;
        uint32      seg_offset;
        XLogRecPtr  result;
    
        fullsegs = bytepos / UsableBytesInSegment;
        bytesleft = bytepos % UsableBytesInSegment;
    
        if (bytesleft < XLOG_BLCKSZ - SizeOfXLogLongPHD)
        {
            //剩余的字节数 < XLOG_BLCKSZ - SizeOfXLogLongPHD    
            /* fits on first page of segment */
            //填充在segment的第一个page中
            seg_offset = bytesleft + SizeOfXLogLongPHD;
        }
        else
        {
            //剩余的字节数 >= XLOG_BLCKSZ - SizeOfXLogLongPHD    
            /* account for the first page on segment with long header */
            //在segment中说明long header
            seg_offset = XLOG_BLCKSZ;
            bytesleft -= XLOG_BLCKSZ - SizeOfXLogLongPHD;
    
            fullpages = bytesleft / UsableBytesInPage;
            bytesleft = bytesleft % UsableBytesInPage;
    
            seg_offset += fullpages * XLOG_BLCKSZ + bytesleft + SizeOfXLogShortPHD;
        }
    
        XLogSegNoOffsetToRecPtr(fullsegs, seg_offset, wal_segment_size, result);
    
        return result;
    }
    
    /* The number of bytes in a WAL segment usable for WAL data. */
    //WAL segment file中可用于WAL data的字节数(不包括page header)
    static int UsableBytesInSegment;
    
    

**CopyXLogRecordToWAL**  
CopyXLogRecordToWAL是XLogInsertRecord中的子过程,用于拷贝XLOG Record到WAL中的保留区域.

    
    
    /*
     * Subroutine of XLogInsertRecord.  Copies a WAL record to an already-reserved
     * area in the WAL.
     * XLogInsertRecord中的子过程.
     * 拷贝XLOG Record到WAL中的保留区域.
     */
    static void
    CopyXLogRecordToWAL(int write_len, bool isLogSwitch, XLogRecData *rdata,
                        XLogRecPtr StartPos, XLogRecPtr EndPos)
    {
        char       *currpos;//当前指针位置
        int         freespace;//空闲空间
        int         written;//已写入的大小
        XLogRecPtr  CurrPos;//事务日志位置
        XLogPageHeader pagehdr;//Page Header
    
        /*
         * Get a pointer to the right place in the right WAL buffer to start
         * inserting to.
         * 在合适的WAL buffer中获取指针用于确定插入的位置
         */
        CurrPos = StartPos;//赋值为开始位置
        currpos = GetXLogBuffer(CurrPos);//获取buffer指针
        freespace = INSERT_FREESPACE(CurrPos);//获取空闲空间大小
    
        /*
         * there should be enough space for at least the first field (xl_tot_len)
         * on this page.
         * 在该页上最起码有第一个字段(xl_tot_len)的存储空间
         */
        Assert(freespace >= sizeof(uint32));
    
        /* Copy record data */
        //拷贝记录数据
        written = 0;
        while (rdata != NULL)//循环
        {
            char       *rdata_data = rdata->data;//指针
            int         rdata_len = rdata->len;//大小
    
            while (rdata_len > freespace)//循环
            {
                /*
                 * Write what fits on this page, and continue on the next page.
                 * 该页能写多少就写多少,写不完就继续下一页.
                 */
                //确保最起码剩余SizeOfXLogShortPHD的头部数据存储空间
                Assert(CurrPos % XLOG_BLCKSZ >= SizeOfXLogShortPHD || freespace == 0);
                //内存拷贝
                memcpy(currpos, rdata_data, freespace);
                //指针调整
                rdata_data += freespace;
                //大小调整
                rdata_len -= freespace;
                //写入大小调整
                written += freespace;
                //当前位置调整
                CurrPos += freespace;
    
                /*
                 * Get pointer to beginning of next page, and set the xlp_rem_len
                 * in the page header. Set XLP_FIRST_IS_CONTRECORD.
                 * 获取下一页的开始指针,并在下一页的header中设置xlp_rem_len.
                 * 同时设置XLP_FIRST_IS_CONTRECORD标记.
                 *
                 * It's safe to set the contrecord flag and xlp_rem_len without a
                 * lock on the page. All the other flags were already set when the
                 * page was initialized, in AdvanceXLInsertBuffer, and we're the
                 * only backend that needs to set the contrecord flag.
                 * 就算不持有页锁,设置contrecord标记和xlp_rem_len也是安全的.
                 * 在页面初始化的时候,所有其他标记已通过AdvanceXLInsertBuffer函数初始化,
                 *   我们是需要设置contrecord标记的唯一一个后台进程,不会有其他进程了.
                 */
                currpos = GetXLogBuffer(CurrPos);//获取buffer
                pagehdr = (XLogPageHeader) currpos;//获取page header
                pagehdr->xlp_rem_len = write_len - written;//设置xlp_rem_len
                pagehdr->xlp_info |= XLP_FIRST_IS_CONTRECORD;//设置标记
    
                /* skip over the page header */
                //跳过page header
                if (XLogSegmentOffset(CurrPos, wal_segment_size) == 0)//第一个page
                {
                    CurrPos += SizeOfXLogLongPHD;//Long Header
                    currpos += SizeOfXLogLongPHD;
                }
                else
                {
                    CurrPos += SizeOfXLogShortPHD;//不是第一个page,Short Header
                    currpos += SizeOfXLogShortPHD;
                }
                freespace = INSERT_FREESPACE(CurrPos);//获取空闲空间
            }
            //再次验证
            Assert(CurrPos % XLOG_BLCKSZ >= SizeOfXLogShortPHD || rdata_len == 0);
            //内存拷贝(这时候rdata_len <= freespace)
            memcpy(currpos, rdata_data, rdata_len);
            currpos += rdata_len;//调整指针
            CurrPos += rdata_len;//调整指针
            freespace -= rdata_len;//减少空闲空间
            written += rdata_len;//调整已写入大小
    
            rdata = rdata->next;//下一批数据
        }
        Assert(written == write_len);//确保已写入 == 需写入大小
    
        /*
         * If this was an xlog-switch, it's not enough to write the switch record,
         * we also have to consume all the remaining space in the WAL segment.  We
         * have already reserved that space, but we need to actually fill it.
         * 如果是xlog-switch并且没有足够的空间写切换的记录,
         *   这时候不得不消费WAL segment剩余的空间.
         * 我们已经预留了空间,但需要执行实际的填充.
         */
        if (isLogSwitch && XLogSegmentOffset(CurrPos, wal_segment_size) != 0)
        {
            /* An xlog-switch record doesn't contain any data besides the header */
            //在header后,xlog-switch没有包含任何数据.
            Assert(write_len == SizeOfXLogRecord);
    
            /* Assert that we did reserve the right amount of space */
            //验证预留了合适的空间
            Assert(XLogSegmentOffset(EndPos, wal_segment_size) == 0);
    
            /* Use up all the remaining space on the current page */
            //在当前页面使用所有的剩余空间
            CurrPos += freespace;
    
            /*
             * Cause all remaining pages in the segment to be flushed, leaving the
             * XLog position where it should be, at the start of the next segment.
             * We do this one page at a time, to make sure we don't deadlock
             * against ourselves if wal_buffers < wal_segment_size.
             * 由于该segment中所有剩余pages将被刷出,把XLog位置指向下一个segment的开始.
             * 一个page我们只做一次,在wal_buffers < wal_segment_size的情况下,
             *   确保我们自己不会出现死锁.
             */
            while (CurrPos < EndPos)//循环
            {
                /*
                 * The minimal action to flush the page would be to call
                 * WALInsertLockUpdateInsertingAt(CurrPos) followed by
                 * AdvanceXLInsertBuffer(...).  The page would be left initialized
                 * mostly to zeros, except for the page header (always the short
                 * variant, as this is never a segment's first page).
                 * 刷出page的最小化动作是:调用WALInsertLockUpdateInsertingAt(CurrPos)
                 *   然后接着调用AdvanceXLInsertBuffer(...).
                 * 除了page header(通常为short格式,除了segment的第一个page)外,其余部分均初始化为ascii 0.
                 * 
                 * The large vistas of zeros are good for compressibility, but the
                 * headers interrupting them every XLOG_BLCKSZ (with values that
                 * differ from page to page) are not.  The effect varies with
                 * compression tool, but bzip2 for instance compresses about an
                 * order of magnitude worse if those headers are left in place.
                 * 连续的ascii 0非常适合压缩,但每个page的头部数据(用于分隔page&page)把这些0隔开了.
                 * 这种效果随压缩工具的不同而不同，但是如果保留这些头文件，则bzip2的压缩效果会差一个数量级。
                 *
                 * Rather than complicating AdvanceXLInsertBuffer itself (which is
                 * called in heavily-loaded circumstances as well as this lightly-                 * loaded one) with variant behavior, we just use GetXLogBuffer
                 * (which itself calls the two methods we need) to get the pointer
                 * and zero most of the page.  Then we just zero the page header.
                 * 与其让AdvanceXLInsertBuffer本身(在重载环境和这个负载较轻的环境中调用)变得复杂，
                 *  不如使用GetXLogBuffer(调用了我们需要的两个方法)来初始化page(初始化为ascii 0)/
                 * 然后把page header设置为ascii 0.
                 */
                currpos = GetXLogBuffer(CurrPos);//获取buffer
                MemSet(currpos, 0, SizeOfXLogShortPHD);//设置头部为ascii 0
    
                CurrPos += XLOG_BLCKSZ;//修改指针
            }
        }
        else
        {
            /* Align the end position, so that the next record starts aligned */
            //对齐末尾位置,以便下一个记录可以从对齐的位置开始
            CurrPos = MAXALIGN64(CurrPos);
        }
    
        if (CurrPos != EndPos)//验证
            elog(PANIC, "space reserved for WAL record does not match what was written");
    }
    
    

### 三、跟踪分析

测试脚本如下:

    
    
    drop table t_wal_longtext;
    create table t_wal_longtext(c1 int not null,c2  varchar(3000),c3 varchar(3000),c4 varchar(3000));
    insert into t_wal_longtext(c1,c2,c3,c4) 
    select i,rpad('C2-'||i,3000,'2'),rpad('C3-'||i,3000,'3'),rpad('C4-'||i,3000,'4') 
    from generate_series(1,7) as i;
    
    

**ReserveXLogInsertLocation**  
插入数据:

    
    
    insert into t_wal_longtext(c1,c2,c3,c4) VALUES(8,'C2-8','C3-8','C4-8');
    
    

设置断点,进入ReserveXLogInsertLocation

    
    
    (gdb) b ReserveXLogInsertLocation
    Breakpoint 1 at 0x54d574: file xlog.c, line 1244.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ReserveXLogInsertLocation (size=74, StartPos=0x7ffebea9d768, EndPos=0x7ffebea9d760, PrevPtr=0x244f4c8)
        at xlog.c:1244
    1244        XLogCtlInsert *Insert = &XLogCtl->Insert;
    (gdb) 
    

输入参数:  
size=74, 这是待插入XLOG Record的大小,其他三个为待设置的值.  
继续执行.  
对齐,74->80(要求为8的N倍,unit64占用8bytes,因此要求8的倍数)

    
    
    (gdb) n
    1249        size = MAXALIGN(size);
    (gdb) 
    1252        Assert(size > SizeOfXLogRecord);
    (gdb) p size
    $1 = 80
    (gdb) 
    

查看插入控制器的信息,其中:  
CurrBytePos = 5498377520,十六进制为0x147BA9530  
PrevBytePos = 5498377464,十六进制为0x147BA94F8  
RedoRecPtr = 5514382312,十六进制为0x148AECBE8 --> 对应pg_control中的Latest checkpoint's
REDO location

    
    
    (gdb) n
    1264        SpinLockAcquire(&Insert->insertpos_lck);
    (gdb) 
    1266        startbytepos = Insert->CurrBytePos;
    (gdb) p *Insert
    $2 = {insertpos_lck = 1 '\001', CurrBytePos = 5498377520, PrevBytePos = 5498377464, pad = '\000' <repeats 127 times>, 
      RedoRecPtr = 5514382312, forcePageWrites = false, fullPageWrites = true, exclusiveBackupState = EXCLUSIVE_BACKUP_NONE, 
      nonExclusiveBackups = 0, lastBackupStart = 0, WALInsertLocks = 0x7f97d1eeb100}
    (gdb) 
    

设置相应的值.  
_值得注意的是插入控制器Insert中的位置信息是不包括page header等信息,是纯粹可用的日志数据,因此数值要比WAL segment
file的数值小._

    
    
    (gdb) n
    1267        endbytepos = startbytepos + size;
    (gdb) 
    1268        prevbytepos = Insert->PrevBytePos;
    (gdb) 
    1269        Insert->CurrBytePos = endbytepos;
    (gdb) 
    1270        Insert->PrevBytePos = startbytepos;
    (gdb) 
    1272        SpinLockRelease(&Insert->insertpos_lck);
    (gdb) 
    

如前所述,需要将“可用字节位置”转换为XLogRecPtr。  
计算实际的开始/结束/上一位置.  
StartPos = 5514538672,0x148B12EB0  
EndPos = 5514538752,0x148B12F00  
PrevPtr = 5514538616,0x148B12E78

    
    
    (gdb) n
    1274        *StartPos = XLogBytePosToRecPtr(startbytepos);
    (gdb) 
    1275        *EndPos = XLogBytePosToEndRecPtr(endbytepos);
    (gdb) 
    1276        *PrevPtr = XLogBytePosToRecPtr(prevbytepos);
    (gdb) 
    1282        Assert(XLogRecPtrToBytePos(*StartPos) == startbytepos);
    (gdb) p *StartPos
    $4 = 5514538672
    (gdb) p *EndPos
    $5 = 5514538752
    (gdb) p *PrevPtr
    $6 = 5514538616
    (gdb) 
    

验证相互转换是没有问题的.

    
    
    (gdb) n
    1283        Assert(XLogRecPtrToBytePos(*EndPos) == endbytepos);
    (gdb) 
    1284        Assert(XLogRecPtrToBytePos(*PrevPtr) == prevbytepos);
    (gdb) 
    1285    }
    (gdb) 
    XLogInsertRecord (rdata=0xf9cc70 <hdr_rdt>, fpw_lsn=5514538520, flags=1 '\001') at xlog.c:1072
    1072            inserted = true;
    (gdb) 
    

DONE!

**CopyXLogRecordToWAL-场景1:不跨WAL page**  
测试脚本如下:

    
    
    insert into t_wal_longtext(c1,c2,c3,c4) VALUES(8,'C2-8','C3-8','C4-8');
    
    

继续上一条SQL的跟踪.  
设置断点,进入CopyXLogRecordToWAL

    
    
    (gdb) b CopyXLogRecordToWAL
    Breakpoint 3 at 0x54dcdf: file xlog.c, line 1479.
    (gdb) c
    Continuing.
    
    Breakpoint 3, CopyXLogRecordToWAL (write_len=74, isLogSwitch=false, rdata=0xf9cc70 <hdr_rdt>, StartPos=5514538672, 
        EndPos=5514538752) at xlog.c:1479
    1479        CurrPos = StartPos;
    (gdb) 
    

输入参数:  
write_len=74, --> 待写入大小  
isLogSwitch=false, --> 是否日志切换(不需要)  
rdata=0xf9cc70 <\hdr_rdt>, --> 需写入的数据地址  
StartPos=5514538672, --> 开始位置  
EndPos=5514538752 --> 结束位置

    
    
    (gdb) n
    1480        currpos = GetXLogBuffer(CurrPos);
    (gdb) 
    

在合适的WAL buffer中获取指针用于确定插入的位置.  
进入函数GetXLogBuffer,输入参数ptr为5514538672,即开始位置.

    
    
    (gdb) step
    GetXLogBuffer (ptr=5514538672) at xlog.c:1854
    1854        if (ptr / XLOG_BLCKSZ == cachedPage)
    (gdb) p ptr / 8192 --> 取模
    $7 = 673161
    (gdb) 
    (gdb) p cachedPage
    $8 = 673161
    (gdb) 
    

GetXLogBuffer->ptr / XLOG_BLCKSZ == cachedPage,进入相应的处理逻辑  
**注意:cachedPage是静态变量,具体在哪个地方赋值,后续需再行分析**

    
    
    (gdb) n
    1856            Assert(((XLogPageHeader) cachedPos)->xlp_magic == XLOG_PAGE_MAGIC);
    (gdb) 
    1857            Assert(((XLogPageHeader) cachedPos)->xlp_pageaddr == ptr - (ptr % XLOG_BLCKSZ));
    (gdb) 
    1858            return cachedPos + ptr % XLOG_BLCKSZ;
    

GetXLogBuffer->cachedPos开头是XLogPageHeader结构体

    
    
    (gdb) p *((XLogPageHeader) cachedPos)
    $14 = {xlp_magic = 53400, xlp_info = 5, xlp_tli = 1, xlp_pageaddr = 5514534912, xlp_rem_len = 71}
    (gdb) 
    (gdb) x/24bx (0x7f97d29fe000)
    0x7f97d29fe000: 0x98    0xd0    0x05    0x00    0x01    0x00    0x00    0x00
    0x7f97d29fe008: 0x00    0x20    0xb1    0x48    0x01    0x00    0x00    0x00
    0x7f97d29fe010: 0x47    0x00    0x00    0x00    0x00    0x00    0x00    0x00
    

回到CopyXLogRecordToWAL,buffer的地址为0x7f97d29feeb0

    
    
    (gdb) n
    1945    }
    (gdb) 
    CopyXLogRecordToWAL (write_len=74, isLogSwitch=false, rdata=0xf9cc70 <hdr_rdt>, StartPos=5514538672, EndPos=5514538752)
        at xlog.c:1481
    1481        freespace = INSERT_FREESPACE(CurrPos);
    (gdb) 
    (gdb) p currpos
    $16 = 0x7f97d29feeb0 ""
    (gdb) 
    

计算空闲空间,确保在该页上最起码有第一个字段(xl_tot_len)的存储空间(4字节).

    
    
    (gdb) n
    1487        Assert(freespace >= sizeof(uint32));
    (gdb) p freespace
    $21 = 4432
    (gdb) 
    

开始拷贝记录数据.

    
    
    (gdb) n
    1490        written = 0; --> 记录已写入的大小
    (gdb) 
    1491        while (rdata != NULL)
    

**rdata的分析详见第四部分** ,继续执行

    
    
    (gdb) n
    1493            char       *rdata_data = rdata->data;
    (gdb) 
    1494            int         rdata_len = rdata->len;
    (gdb) 
    1496            while (rdata_len > freespace)
    (gdb) p rdata_len
    $34 = 46
    (gdb) p freespace
    $35 = 4432
    (gdb) 
    

rdata_len < freespace,无需进入子循环.  
再次进行验证没有问题,执行内存拷贝.

    
    
    (gdb) n
    1536            Assert(CurrPos % XLOG_BLCKSZ >= SizeOfXLogShortPHD || rdata_len == 0);
    (gdb) 
    1537            memcpy(currpos, rdata_data, rdata_len);
    (gdb) 
    1538            currpos += rdata_len;
    (gdb) 
    1539            CurrPos += rdata_len;
    (gdb) 
    1540            freespace -= rdata_len;
    (gdb) 
    1541            written += rdata_len;
    (gdb) 
    1543            rdata = rdata->next;
    (gdb) 
    1491        while (rdata != NULL)
    (gdb) p currpos
    $36 = 0x7f97d29feede ""
    (gdb) p CurrPos
    $37 = 5514538718
    (gdb) p freespace
    $38 = 4386
    (gdb) p written
    $39 = 46
    (gdb) 
    

rdata共有四部分,继续写入第二/三/四部分.

    
    
    ...
    1491        while (rdata != NULL)
    (gdb) 
    1493            char       *rdata_data = rdata->data;
    (gdb) 
    1494            int         rdata_len = rdata->len;
    (gdb) 
    1496            while (rdata_len > freespace)
    (gdb) 
    1536            Assert(CurrPos % XLOG_BLCKSZ >= SizeOfXLogShortPHD || rdata_len == 0);
    (gdb) 
    1537            memcpy(currpos, rdata_data, rdata_len);
    (gdb) 
    1538            currpos += rdata_len;
    (gdb) 
    1539            CurrPos += rdata_len;
    (gdb) 
    1540            freespace -= rdata_len;
    (gdb) 
    1541            written += rdata_len;
    (gdb) 
    1543            rdata = rdata->next;
    (gdb) 
    1491        while (rdata != NULL)
    (gdb) 
    

完成写入74bytes

    
    
    (gdb) 
    1545        Assert(written == write_len);
    (gdb) p written
    $40 = 74
    (gdb) 
    

无需执行日志切换的相关操作.  
对齐CurrPos

    
    
    (gdb) n
    1552        if (isLogSwitch && XLogSegmentOffset(CurrPos, wal_segment_size) != 0)
    (gdb) 
    1599            CurrPos = MAXALIGN64(CurrPos);
    (gdb) p CurrPos
    $41 = 5514538746
    (gdb) n
    1602        if (CurrPos != EndPos)
    (gdb) p CurrPos
    $42 = 5514538752
    (gdb) 
    (gdb) p 5514538746 % 8
    $44 = 2 --> 需补6个字节,5514538746 --> 5514538752
    

对齐后,CurrPos == EndPos,否则报错!

    
    
    (gdb) p EndPos
    $45 = 5514538752
    

结束调用

    
    
    (gdb) n
    1604    }
    (gdb) 
    XLogInsertRecord (rdata=0xf9cc70 <hdr_rdt>, fpw_lsn=5514538520, flags=1 '\001') at xlog.c:1098
    1098            if ((flags & XLOG_MARK_UNIMPORTANT) == 0)
    (gdb) 
    

DONE!

**CopyXLogRecordToWAL-场景2:跨WAL page 后续再行分析**

### 四、再论WAL Record

在内存中,WAL Record通过rdata存储,该变量其实是全局静态变量hdr_rdt,类型为XLogRecData,XLOG
Record通过XLogRecData链表组织起来( **这个设计很赞,写入无需理会结构,按链表逐个写数据即可** ).  
rdata由4部分组成:  
第一部分是XLogRecord + XLogRecordBlockHeader + XLogRecordDataHeaderShort,共46字节  
第二部分是xl_heap_header,5个字节  
第三部分是tuple data,20个字节  
第四部分是xl_heap_insert,3个字节

    
    
    ------------------------------------------------------------------- 1
    (gdb) p *rdata 
    $22 = {next = 0x244f2c0, data = 0x244f4c0 "J", len = 46} 
    (gdb) p *(XLogRecord *)rdata->data --> XLogRecord
    $27 = {xl_tot_len = 74, xl_xid = 2268, xl_prev = 5514538616, xl_info = 0 '\000', xl_rmid = 10 '\n', xl_crc = 1158677949}
    (gdb) p *(XLogRecordBlockHeader *)(0x244f4c0+24) --> XLogRecordBlockHeader
    $29 = {id = 0 '\000', fork_flags = 32 ' ', data_length = 25}
    (gdb) x/2bx (0x244f4c0+44) --> XLogRecordDataHeaderShort
    0x244f4ec:  0xff    0x03
    ------------------------------------------------------------------- 2 
    (gdb) p *rdata->next
    $23 = {next = 0x244f2d8, data = 0x7ffebea9d830 "\004", len = 5}
    (gdb) p *(xl_heap_header *)rdata->next->data
    $32 = {t_infomask2 = 4, t_infomask = 2050, t_hoff = 24 '\030'}
    ------------------------------------------------------------------- 3
    (gdb) p *rdata->next->next
    $24 = {next = 0x244f2a8, data = 0x24e6a2f "", len = 20}
    (gdb) x/20bc  0x24e6a2f
    0x24e6a2f:  0 '\000'    8 '\b'  0 '\000'    0 '\000'    0 '\000'    11 '\v' 67 'C'  50 '2'
    0x24e6a37:  45 '-'  56 '8'  11 '\v' 67 'C'  51 '3'  45 '-'  56 '8'  11 '\v'
    0x24e6a3f:  67 'C'  52 '4'  45 '-'  56 '8'
    (gdb) 
    ------------------------------------------------------------------- 4
    (gdb) p *rdata->next->next->next
    $25 = {next = 0x0, data = 0x7ffebea9d840 "\b", len = 3}
    (gdb) 
    (gdb) p *(xl_heap_insert *)rdata->next->next->next->data
    $33 = {offnum = 8, flags = 0 '\000'}
    

### 五、参考资料

[PostgreSQL 源码解读（4）-插入数据#3（heap_insert）](https://www.jianshu.com/p/0e16fce4ca73)  
[PostgreSQL 事务日志WAL结构浅析](https://www.jianshu.com/p/b9087d9f20e2)  
[PostgreSQL 源码解读（110）- WAL#6（Insert&WAL -XLogRecordAssemble记录组装函数）](https://www.jianshu.com/p/ded2ea311e82)  
[PostgreSQL 源码解读（111）- WAL#7（Insert&WAL - XLogRecordAssemble-FPW）](https://www.jianshu.com/p/2c6c29a01eda)  
[PostgreSQL 源码解读（112）-WAL#8（XLogCtrl数据结构）](https://www.jianshu.com/p/69323c1c9994)  
[PG Source Code](https://doxygen.postgresql.org)

