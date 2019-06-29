本节简单介绍了XLogRecordAssemble函数的实现逻辑,该函数从已注册的数据和缓冲区中组装XLOG
record到XLogRecData链中,为XLOG Record的插入作准备。

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
    

### 二、源码解读

XLogRecordAssemble函数从已注册的数据和缓冲区中组装XLOG
record到XLogRecData链中,组装完成后可以使用XLogInsertRecord()函数插入到WAL buffer中.

    
    
     /*
      * Assemble a WAL record from the registered data and buffers into an
      * XLogRecData chain, ready for insertion with XLogInsertRecord().
      * 从已注册的数据和缓冲区中组装XLOG record到XLogRecData链中,
      * 组装完成后可以使用XLogInsertRecord()函数插入.
      * 
      * The record header fields are filled in, except for the xl_prev field. The
      * calculated CRC does not include the record header yet.
      * 除了xl_prev外,XLOG Record的header域已填充完毕.
      * 计算的CRC还没有包含header信息.
      *
      * If there are any registered buffers, and a full-page image was not taken
      * of all of them, *fpw_lsn is set to the lowest LSN among such pages. This
      * signals that the assembled record is only good for insertion on the
      * assumption that the RedoRecPtr and doPageWrites values were up-to-date.
      * 如存在已注册的缓冲区,而且full-page-image没有全部包括这些数据,
      *   *fpw_lsn设置为这些页面中最小的LSN.
      * 基于RedoRecPtr和doPageWrites已更新为最新的假设,
      *   已组装的XLOG Record对在此假设上的插入是OK的.
      */
     static XLogRecData *
     XLogRecordAssemble(RmgrId rmid, uint8 info,
                        XLogRecPtr RedoRecPtr, bool doPageWrites,
                        XLogRecPtr *fpw_lsn)
    {
         XLogRecData *rdt;//XLogRecData指针
         uint32      total_len = 0;//XLOG Record大小
         int         block_id;//块ID
         pg_crc32c   rdata_crc;//CRC
         registered_buffer *prev_regbuf = NULL;//已注册的buffer指针
         XLogRecData *rdt_datas_last;//
         XLogRecord *rechdr;//头部信息
         char       *scratch = hdr_scratch;
    
         /*
          * Note: this function can be called multiple times for the same record.
          * All the modifications we do to the rdata chains below must handle that.
          * 对于同一个XLOG Record,该函数可以被多次调用.
          * 下面我们对rdata链进行的所有更新必须处理这种情况.
          */
    
         /* The record begins with the fixed-size header */
         //XLOG Record的头部大小是固定的.
         rechdr = (XLogRecord *) scratch;
         scratch += SizeOfXLogRecord;//指针移动
    
         hdr_rdt.next = NULL;//hdr_rdt --> static XLogRecData hdr_rdt;
         rdt_datas_last = &hdr_rdt;//
         hdr_rdt.data = hdr_scratch;//rmgr数据的起始偏移
    
         /*
          * Enforce consistency checks for this record if user is looking for it.
          * Do this before at the beginning of this routine to give the possibility
          * for callers of XLogInsert() to pass XLR_CHECK_CONSISTENCY directly for
          * a record.
          * 如正在搜索此记录,则强制检查该记录的一致性.
          * 在该处理过程开始前执行此项处理,以便XLogInsert()的调用者
          *   可以直接传递XLR_CHECK_CONSISTENCY给XLOG Record.
          */
         if (wal_consistency_checking[rmid])
             info |= XLR_CHECK_CONSISTENCY;
    
         /*
          * Make an rdata chain containing all the data portions of all block
          * references. This includes the data for full-page images. Also append
          * the headers for the block references in the scratch buffer.
          * 构造保存所有块参考的数据部分的rdata链.这包括FPI的数据.
          * 同时,在scratch缓冲区中为所有的块引用追加头部信息.
          */
         *fpw_lsn = InvalidXLogRecPtr;//初始化变量
         for (block_id = 0; block_id < max_registered_block_id; block_id++)//遍历已注册的block
         {
             registered_buffer *regbuf = &registered_buffers[block_id];//获取根据block_id获取缓冲区
             bool        needs_backup;//是否需要backup block
             bool        needs_data;//是否需要data
             XLogRecordBlockHeader bkpb;//XLogRecordBlockHeader
             XLogRecordBlockImageHeader bimg;//XLogRecordBlockImageHeader
             XLogRecordBlockCompressHeader cbimg = {0};//压缩存储时需要
             bool        samerel;//是否同一个rel?
             bool        is_compressed = false;//是否压缩
             bool        include_image;//是否包括FPI
    
             if (!regbuf->in_use)//未在使用,继续下一个
                 continue;
    
             /* Determine if this block needs to be backed up */
             //确定此block是否需要backup
             if (regbuf->flags & REGBUF_FORCE_IMAGE)
                 needs_backup = true;//强制要求FPI
             else if (regbuf->flags & REGBUF_NO_IMAGE)
                 needs_backup = false;//强制要求不要IMAGE
             else if (!doPageWrites)
                 needs_backup = false;//doPageWrites标记设置为F
             else//doPageWrites = T
             {
                 /*
                  * We assume page LSN is first data on *every* page that can be
                  * passed to XLogInsert, whether it has the standard page layout
                  * or not.
                  * 不管该page是否标准page layout,
                  *   我们假定在每一个page中最前面的数据是page LSN,
                  *   
                  */
                 XLogRecPtr  page_lsn = PageGetLSN(regbuf->page);//获取LSN
    
                 needs_backup = (page_lsn <= RedoRecPtr);//是否需要backup
                 if (!needs_backup)//不需要
                 {
                     if (*fpw_lsn == InvalidXLogRecPtr || page_lsn < *fpw_lsn)
                         *fpw_lsn = page_lsn;//设置LSN
                 }
             }
    
             /* Determine if the buffer data needs to included */
             //确定buffer中的data是否需要包括在其中
             if (regbuf->rdata_len == 0)//没有数据
                 needs_data = false;
             else if ((regbuf->flags & REGBUF_KEEP_DATA) != 0)//需要包括data
                 needs_data = true;
             else
                 needs_data = !needs_backup;//needs_backup取反
             //BlockHeader设置值
             bkpb.id = block_id;//块ID
             bkpb.fork_flags = regbuf->forkno;//forkno
             bkpb.data_length = 0;//数据长度
    
             if ((regbuf->flags & REGBUF_WILL_INIT) == REGBUF_WILL_INIT)
                 bkpb.fork_flags |= BKPBLOCK_WILL_INIT;//设置标记
    
             /*
              * If needs_backup is true or WAL checking is enabled for current
              * resource manager, log a full-page write for the current block.
              * 如needs_backup为T,或者当前RM的WAL检查已启用,
              *   为当前block执行full-page-write
              */
             //需要backup或者要求执行一致性检查  
             include_image = needs_backup || (info & XLR_CHECK_CONSISTENCY) != 0;
    
             if (include_image)
             {
                 //包含块镜像
                 Page        page = regbuf->page;//获取对应的page
                 uint16      compressed_len = 0;//压缩后的大小
    
                 /*
                  * The page needs to be backed up, so calculate its hole length
                  * and offset.
                  * page需要备份,计算空闲空间大小和偏移
                  */
                 if (regbuf->flags & REGBUF_STANDARD)
                 {
                     //如为标准的REGBUF
                     /* Assume we can omit data between pd_lower and pd_upper */
                     //假定我们可以省略pd_lower和pd_upper之间的数据
                     uint16      lower = ((PageHeader) page)->pd_lower;//获取lower
                     uint16      upper = ((PageHeader) page)->pd_upper;//获取upper
    
                     if (lower >= SizeOfPageHeaderData &&
                         upper > lower &&
                         upper <= BLCKSZ)
                     {
                         //lower大于Page的头部 && upper大于lower && upper小于块大小
                         bimg.hole_offset = lower;
                         cbimg.hole_length = upper - lower;
                     }
                     else
                     {
                         /* No "hole" to remove */
                         //没有空闲空间可以移除
                         bimg.hole_offset = 0;
                         cbimg.hole_length = 0;
                     }
                 }
                 else
                 {
                     //不是标准的REGBUF
                     /* Not a standard page header, don't try to eliminate "hole" */
                     //不是标准的page header,不要尝试估算"hole"
                     bimg.hole_offset = 0;
                     cbimg.hole_length = 0;
                 }
    
                 /*
                  * Try to compress a block image if wal_compression is enabled
                  * 如果wal_compression启用,则尝试压缩
                  */
                 if (wal_compression)
                 {
                     is_compressed =
                         XLogCompressBackupBlock(page, bimg.hole_offset,
                                                 cbimg.hole_length,
                                                 regbuf->compressed_page,
                                                 &compressed_len);//调用XLogCompressBackupBlock压缩
                 }
    
                 /*
                  * Fill in the remaining fields in the XLogRecordBlockHeader
                  * struct
                  * 填充XLogRecordBlockHeader结构体的剩余域字段
                  */
                 bkpb.fork_flags |= BKPBLOCK_HAS_IMAGE;
    
                 /*
                  * Construct XLogRecData entries for the page content.
                  * 为page内容构造XLogRecData入口
                  */
                 rdt_datas_last->next = &regbuf->bkp_rdatas[0];
                 rdt_datas_last = rdt_datas_last->next;
                 //设置标记
                 bimg.bimg_info = (cbimg.hole_length == 0) ? 0 : BKPIMAGE_HAS_HOLE;
    
                 /*
                  * If WAL consistency checking is enabled for the resource manager
                  * of this WAL record, a full-page image is included in the record
                  * for the block modified. During redo, the full-page is replayed
                  * only if BKPIMAGE_APPLY is set.
                  * 如WAL一致性检查已启用,被更新的block已在XLOG Record中包含了FPI.
                  * 在redo期间,在设置了BKPIMAGE_APPLY标记的情况下full-page才会回放.
                  */
                 if (needs_backup)
                     bimg.bimg_info |= BKPIMAGE_APPLY;//设置标记
    
                 if (is_compressed)//是否压缩?
                 {
                     bimg.length = compressed_len;//压缩后的空间
                     bimg.bimg_info |= BKPIMAGE_IS_COMPRESSED;//压缩标记
    
                     rdt_datas_last->data = regbuf->compressed_page;//放在registered_buffer中
                     rdt_datas_last->len = compressed_len;//长度
                 }
                 else
                 {
                     //没有压缩
                     //image的大小
                     bimg.length = BLCKSZ - cbimg.hole_length;
    
                     if (cbimg.hole_length == 0)
                     {
                         rdt_datas_last->data = page;//数据指针直接指向page
                         rdt_datas_last->len = BLCKSZ;//大小为block size
                     }
                     else
                     {
                         /* must skip the hole */
                         //跳过hole
                         rdt_datas_last->data = page;//数据指针
                         rdt_datas_last->len = bimg.hole_offset;//获取hole的偏移
    
                         rdt_datas_last->next = &regbuf->bkp_rdatas[1];//第2部分
                         rdt_datas_last = rdt_datas_last->next;//
    
                         rdt_datas_last->data =
                             page + (bimg.hole_offset + cbimg.hole_length);//指针指向第二部分
                         rdt_datas_last->len =
                             BLCKSZ - (bimg.hole_offset + cbimg.hole_length);//设置长度
                     }
                 }
    
                 total_len += bimg.length;//调整总长度
             }
    
             if (needs_data)//需要包含数据
             {
                 /*
                  * Link the caller-supplied rdata chain for this buffer to the
                  * overall list.
                  * 把该缓冲区链接到调用者提供的rdata链中构成一个整体的链表
                  */
                 bkpb.fork_flags |= BKPBLOCK_HAS_DATA;//设置标记
                 bkpb.data_length = regbuf->rdata_len;//长度
                 total_len += regbuf->rdata_len;//总大小
    
                 rdt_datas_last->next = regbuf->rdata_head;//调整指针
                 rdt_datas_last = regbuf->rdata_tail;
             }
    
             //存在上一个regbuf 而且是同一个RefFileNode(关系一样/表空间一样/block一样)
             if (prev_regbuf && RelFileNodeEquals(regbuf->rnode, prev_regbuf->rnode))
             {
                 samerel = true;//设置标记
                 bkpb.fork_flags |= BKPBLOCK_SAME_REL;//同一个REL
             }
             else
                 samerel = false;
             prev_regbuf = regbuf;//切换为当前的regbuf
    
             /* Ok, copy the header to the scratch buffer */
             //已OK,拷贝头部信息到scratch缓冲区中
             memcpy(scratch, &bkpb, SizeOfXLogRecordBlockHeader);
             scratch += SizeOfXLogRecordBlockHeader;//调整偏移
             if (include_image)
             {
                 //包含FPI,追加SizeOfXLogRecordBlockImageHeader
                 memcpy(scratch, &bimg, SizeOfXLogRecordBlockImageHeader);
                 scratch += SizeOfXLogRecordBlockImageHeader;//调整偏移
                 if (cbimg.hole_length != 0 && is_compressed)
                 {
                     //压缩存储,追加SizeOfXLogRecordBlockCompressHeader
                     memcpy(scratch, &cbimg,
                            SizeOfXLogRecordBlockCompressHeader);
                     scratch += SizeOfXLogRecordBlockCompressHeader;//调整偏移
                 }
             }
             if (!samerel)
             {
                 //不是同一个REL,追加RelFileNode
                 memcpy(scratch, &regbuf->rnode, sizeof(RelFileNode));
                 scratch += sizeof(RelFileNode);//调整偏移
             }
             //后跟BlockNumber
             memcpy(scratch, &regbuf->block, sizeof(BlockNumber));
             scratch += sizeof(BlockNumber);//调整偏移
         }
    
         /* followed by the record's origin, if any */
         //接下来,是XLOG Record origin
         if ((curinsert_flags & XLOG_INCLUDE_ORIGIN) &&
             replorigin_session_origin != InvalidRepOriginId)
         {
             //
             *(scratch++) = (char) XLR_BLOCK_ID_ORIGIN;
             memcpy(scratch, &replorigin_session_origin, sizeof(replorigin_session_origin));
             scratch += sizeof(replorigin_session_origin);
         }
    
         /* followed by main data, if any */
         //接下来是main data
         if (mainrdata_len > 0)//main data大小 > 0
         {
             if (mainrdata_len > 255)//超过255,则使用Long格式
             {
                 *(scratch++) = (char) XLR_BLOCK_ID_DATA_LONG;
                 memcpy(scratch, &mainrdata_len, sizeof(uint32));
                 scratch += sizeof(uint32);
             }
             else//否则使用Short格式
             {
                 *(scratch++) = (char) XLR_BLOCK_ID_DATA_SHORT;
                 *(scratch++) = (uint8) mainrdata_len;
             }
             rdt_datas_last->next = mainrdata_head;
             rdt_datas_last = mainrdata_last;
             total_len += mainrdata_len;
         }
         rdt_datas_last->next = NULL;
    
         hdr_rdt.len = (scratch - hdr_scratch);//头部大小
         total_len += hdr_rdt.len;//总大小
    
         /*
          * Calculate CRC of the data
          * 计算数据的CRC
          *
          * Note that the record header isn't added into the CRC initially since we
          * don't know the prev-link yet.  Thus, the CRC will represent the CRC of
          * the whole record in the order: rdata, then backup blocks, then record
          * header.
          * 由于我们还不知道prev-link的数值,因此头部不在最初的CRC中.
          * 因此，CRC将按照以下顺序表示整个记录的CRC: rdata，然后是backup blocks，然后是record header。
          */
         INIT_CRC32C(rdata_crc);
         COMP_CRC32C(rdata_crc, hdr_scratch + SizeOfXLogRecord, hdr_rdt.len - SizeOfXLogRecord);
         for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
             COMP_CRC32C(rdata_crc, rdt->data, rdt->len);
    
         /*
          * Fill in the fields in the record header. Prev-link is filled in later,
          * once we know where in the WAL the record will be inserted. The CRC does
          * not include the record header yet.
          * 填充记录头部信息的其他域字段.
          * Prev-link将在该记录插入在哪里的时候再填充.
          * CRC还不包括记录的头部信息.
          */
         rechdr->xl_xid = GetCurrentTransactionIdIfAny();
         rechdr->xl_tot_len = total_len;
         rechdr->xl_info = info;
         rechdr->xl_rmid = rmid;
         rechdr->xl_prev = InvalidXLogRecPtr;
         rechdr->xl_crc = rdata_crc;
    
         return &hdr_rdt;
    }
    
    

### 三、跟踪分析

**场景一:清除数据后,执行checkpoint后的第一次插入**  
测试脚本如下:

    
    
    testdb=# truncate table t_wal_partition;
    TRUNCATE TABLE
    testdb=# checkpoint;
    CHECKPOINT
    testdb=# insert into t_wal_partition(c1,c2,c3) VALUES(1,'checkpoint','checkpoint');
    
    

设置断点,进入XLogRecordAssemble

    
    
    (gdb) b XLogRecordAssemble
    Breakpoint 1 at 0x565411: file xloginsert.c, line 488.
    (gdb) c
    Continuing.
    
    Breakpoint 1, XLogRecordAssemble (rmid=10 '\n', info=128 '\200', RedoRecPtr=5507633240, doPageWrites=true, 
        fpw_lsn=0x7fff05cfe378) at xloginsert.c:488
    488     uint32      total_len = 0;
    

输入参数:  
rmid=10即0x0A --> Heap  
RedoRecPtr=5507633240  
doPageWrites=true,需要full-page-write  
fpw_lsn=0x7fff05cfe378  
接下来是变量赋值,  
其中hdr_scratch的定义为:static char *hdr_scratch = NULL;  
hdr_rdt的定义为:static XLogRecData hdr_rdt;

    
    
    (gdb) n
    491     registered_buffer *prev_regbuf = NULL;
    (gdb) 
    494     char       *scratch = hdr_scratch;
    (gdb) 
    502     rechdr = (XLogRecord *) scratch;
    (gdb) 
    503     scratch += SizeOfXLogRecord;
    (gdb) 
    

XLOG Record的头部信息

    
    
    (gdb) p *(XLogRecord *)rechdr
    $11 = {xl_tot_len = 114, xl_xid = 1997, xl_prev = 5507669824, xl_info = 128 '\200', xl_rmid = 1 '\001', xl_crc = 3794462175}
    

scratch指针指向Header之后的地址

    
    
    (gdb) p hdr_scratch
    $12 = 0x18a24c0 "r"
    

为全局变量hdr_rdt赋值

    
    
    505     hdr_rdt.next = NULL;
    (gdb) 
    506     rdt_datas_last = &hdr_rdt;
    (gdb) 
    507     hdr_rdt.data = hdr_scratch;
    (gdb) p hdr_rdt
    $5 = {next = 0x0, data = 0x18a24c0 "r", len = 26}
    (gdb) p *(XLogRecord *)hdr_rdt.data
    $7 = {xl_tot_len = 114, xl_xid = 1997, xl_prev = 5507669824, xl_info = 128 '\200', xl_rmid = 1 '\001', xl_crc = 3794462175}
    

不执行一致性检查

    
    
    (gdb) n
    515     if (wal_consistency_checking[rmid])
    (gdb) 
    523     *fpw_lsn = InvalidXLogRecPtr;
    (gdb) 
    

初始化fpw_lsn,开始循环.  
已注册的block id只有1个.

    
    
    (gdb) n
    524     for (block_id = 0; block_id < max_registered_block_id; block_id++)
    (gdb) p max_registered_block_id
    $13 = 1
    

获取已注册的buffer.  
其中:  
rnode->RelFilenode结构体,spcNode->表空间/dbNode->数据库/relNode->关系  
block->块ID  
page->数据页指针(char *)  
rdata_len->rdata链中的数据总大小  
rdata_head->使用该数据块注册的数据链头  
rdata_tail->使用该数据块注册的数据链尾  
bkp_rdatas->临时rdatas数据引用,用于存储XLogRecordAssemble()中使用的备份块数据.  
**bkp_rdatas用于组装block
image,bkp_rdatas[0]存储空闲空间(hole)前的数据,bkp_rdatas[1]存储空闲空间后的数据.**

    
    
    (gdb) n
    526         registered_buffer *regbuf = &registered_buffers[block_id];
    (gdb) 
    531         XLogRecordBlockCompressHeader cbimg = {0};
    (gdb) p *regbuf
    $14 = {in_use = true, flags = 14 '\016', rnode = {spcNode = 1663, dbNode = 16402, relNode = 25258}, forkno = MAIN_FORKNUM, 
      block = 0, page = 0x7fb8539e7380 "", rdata_len = 32, rdata_head = 0x18a22c0, rdata_tail = 0x18a22d8, bkp_rdatas = {{
          next = 0x18a4230, data = 0x7fb85390f380 "\001", len = 252}, {next = 0x18a22a8, data = 0x7fb85390fe28 "\315\a", 
          len = 5464}}, compressed_page = '\000' <repeats 8195 times>}
    

**注意:  
在内存中,main
data已由函数XLogRegisterData注册,由mainrdata_head和mainrdata_last指针维护,本例中,填充了xl_heap_insert结构体.  
block
data由XLogRegisterBuffer初始化,由XLogRegisterBufData填充数据,在本例中,通过XLogRegisterBufData注册了两次数据,第一次是xl_heap_header结构体,第二次是实际的数据(实质上只是数据指针,最终需要什么数据,由组装器确定).**

    
    
    (gdb) p *mainrdata_head
    $18 = {next = 0x18a22c0, data = 0x7fff05cfe3f0 "\001", len = 3}
    (gdb) p *(xl_heap_insert *)mainrdata_head->data
    $20 = {offnum = 1, flags = 0 '\000'}
    (gdb) p *regbuf->rdata_head
    $32 = {next = 0x18a22d8, data = 0x7fff05cfe3e0 "\003", len = 5}
    (gdb) p *(xl_heap_header *)regbuf->rdata_head->data
    $28 = {t_infomask2 = 3, t_infomask = 2050, t_hoff = 24 '\030'}
    (gdb) p *regbuf->rdata_head->next
    $34 = {next = 0x18a22f0, data = 0x18edaef "", len = 27}
    

以字符格式显示地址0x18edaef之后的27个字节(tuple data)

    
    
    (gdb) x/27bc 0x18edaef
    0x18edaef:  0 '\000'    1 '\001'    0 '\000'    0 '\000'    0 '\000'    23 '\027'   99 'c'  104 'h'
    0x18edaf7:  101 'e' 99 'c'  107 'k' 112 'p' 111 'o' 105 'i' 110 'n' 116 't'
    0x18edaff:  23 '\027'   99 'c'  104 'h' 101 'e' 99 'c'  107 'k' 112 'p' 111 'o'
    0x18edb07:  105 'i' 110 'n' 116 't'
    

继续往下执行,由于该记录是第一条记录,因此无需执行full-page-image

    
    
    (gdb) n
    533         bool        is_compressed = false;
    (gdb) 
    536         if (!regbuf->in_use)
    (gdb) 
    540         if (regbuf->flags & REGBUF_FORCE_IMAGE)
    (gdb) p regbuf->flags
    $36 = 14 '\016'
    (gdb) n
    542         else if (regbuf->flags & REGBUF_NO_IMAGE)
    (gdb) 
    543             needs_backup = false;
    

needs_data为T,事务日志中仅写入tuple data

    
    
    (gdb) n
    564         if (regbuf->rdata_len == 0)
    (gdb) p regbuf->rdata_len
    $37 = 32
    (gdb) n
    566         else if ((regbuf->flags & REGBUF_KEEP_DATA) != 0)
    (gdb) 
    569             needs_data = !needs_backup;
    (gdb) 
    571         bkpb.id = block_id;
    (gdb) p needs_data
    $38 = true
    

设置XLogRecordBlockHeader字段值,page中第一个tuple,标记设置为BKPBLOCK_WILL_INIT

    
    
    (gdb) n
    572         bkpb.fork_flags = regbuf->forkno;
    (gdb) 
    573         bkpb.data_length = 0;
    (gdb) 
    575         if ((regbuf->flags & REGBUF_WILL_INIT) == REGBUF_WILL_INIT)
    (gdb) n
    576             bkpb.fork_flags |= BKPBLOCK_WILL_INIT;
    (gdb) 
    582         include_image = needs_backup || (info & XLR_CHECK_CONSISTENCY) != 0;
    (gdb) p bkpb
    $40 = {id = 0 '\000', fork_flags = 64 '@', data_length = 0}
    (gdb) 
    

不需要执行FPI

    
    
    (gdb) p info
    $41 = 128 '\200'
    (gdb) n
    584         if (include_image)
    (gdb) p include_image
    $42 = false
    (gdb) 
    

需要包含数据

    
    
    (gdb) n
    691         if (needs_data)
    (gdb) 
    697             bkpb.fork_flags |= BKPBLOCK_HAS_DATA;
    (gdb) 
    698             bkpb.data_length = regbuf->rdata_len;
    (gdb) 
    699             total_len += regbuf->rdata_len;
    (gdb) 
    701             rdt_datas_last->next = regbuf->rdata_head;
    (gdb) 
    702             rdt_datas_last = regbuf->rdata_tail;
    (gdb) p bkpb
    $43 = {id = 0 '\000', fork_flags = 96 '`', data_length = 32}
    (gdb) p total_len
    $44 = 32
    (gdb) p *rdt_datas_last
    $45 = {next = 0x18a22c0, data = 0x18a24c0 "r", len = 26}
    

已OK,拷贝头部信息到scratch缓冲区中

    
    
    (gdb) n
    705         if (prev_regbuf && RelFileNodeEquals(regbuf->rnode, prev_regbuf->rnode))
    (gdb) p prev_regbuf
    $46 = (registered_buffer *) 0x0
    (gdb) n
    711             samerel = false;
    (gdb) 
    712         prev_regbuf = regbuf;
    (gdb) 
    715         memcpy(scratch, &bkpb, SizeOfXLogRecordBlockHeader);
    

后面是RefFileNode + BlockNumber

    
    
    (gdb) 
    716         scratch += SizeOfXLogRecordBlockHeader;
    (gdb) 
    717         if (include_image)
    (gdb) 
    728         if (!samerel)
    (gdb) 
    730             memcpy(scratch, &regbuf->rnode, sizeof(RelFileNode));
    (gdb) 
    731             scratch += sizeof(RelFileNode);
    (gdb) 
    733         memcpy(scratch, &regbuf->block, sizeof(BlockNumber));
    (gdb) 
    734         scratch += sizeof(BlockNumber);
    (gdb) 
    524     for (block_id = 0; block_id < max_registered_block_id; block_id++)
    

结束循环

    
    
    524     for (block_id = 0; block_id < max_registered_block_id; block_id++)
    (gdb) 
    

接下来是replorigin_session_origin(实际并不需要)

    
    
    738     if ((curinsert_flags & XLOG_INCLUDE_ORIGIN) &&
    (gdb) p curinsert_flags
    $47 = 1 '\001'
    (gdb) 
    $48 = 1 '\001'
    (gdb) n
    739         replorigin_session_origin != InvalidRepOriginId)
    (gdb) 
    738     if ((curinsert_flags & XLOG_INCLUDE_ORIGIN) &&
    

接下来是main data

    
    
    (gdb) 
    747     if (mainrdata_len > 0)
    (gdb) 
    749         if (mainrdata_len > 255)
    (gdb) 
    757             *(scratch++) = (char) XLR_BLOCK_ID_DATA_SHORT;
    (gdb) 
    (gdb) 
    758             *(scratch++) = (uint8) mainrdata_len;
    (gdb) 
    760         rdt_datas_last->next = mainrdata_head;
    (gdb) 
    761         rdt_datas_last = mainrdata_last;
    (gdb) 
    762         total_len += mainrdata_len;
    (gdb) 
    

计算大小

    
    
    764     rdt_datas_last->next = NULL;
    (gdb) 
    766     hdr_rdt.len = (scratch - hdr_scratch);
    (gdb) p scratch
    $49 = 0x18a24ee ""
    (gdb) p hdr_scratch
    $50 = 0x18a24c0 "r"
    (gdb) p hdr_rdt.len
    $51 = 26
    (gdb) p total_len
    $52 = 35
    (gdb) 
    (gdb) n
    767     total_len += hdr_rdt.len;
    (gdb) 
    

计算CRC

    
    
    (gdb) 
    777     INIT_CRC32C(rdata_crc);
    (gdb) 
    778     COMP_CRC32C(rdata_crc, hdr_scratch + SizeOfXLogRecord, hdr_rdt.len - SizeOfXLogRecord);
    (gdb) 
    779     for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
    (gdb) n
    780         COMP_CRC32C(rdata_crc, rdt->data, rdt->len);
    (gdb) 
    779     for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
    (gdb) 
    780         COMP_CRC32C(rdata_crc, rdt->data, rdt->len);
    (gdb) 
    779     for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
    (gdb) 
    780         COMP_CRC32C(rdata_crc, rdt->data, rdt->len);
    (gdb) 
    779     for (rdt = hdr_rdt.next; rdt != NULL; rdt = rdt->next)
    (gdb) 
    787     rechdr->xl_xid = GetCurrentTransactionIdIfAny();
    

填充记录头部信息的其他域字段.

    
    
    (gdb) n
    788     rechdr->xl_tot_len = total_len;
    (gdb) 
    789     rechdr->xl_info = info;
    (gdb) 
    790     rechdr->xl_rmid = rmid;
    (gdb) 
    791     rechdr->xl_prev = InvalidXLogRecPtr;
    (gdb) 
    792     rechdr->xl_crc = rdata_crc;
    (gdb) 
    794     return &hdr_rdt;
    (gdb) 
    795 }
    (gdb) p rechdr
    $62 = (XLogRecord *) 0x18a24c0
    (gdb) p *rechdr
    $63 = {xl_tot_len = 81, xl_xid = 1998, xl_prev = 0, xl_info = 128 '\200', xl_rmid = 10 '\n', xl_crc = 1852971194}
    (gdb) 
    

**full-page-write场景后续再行分析**

### 四、参考资料

[Write Ahead Logging — WAL](http://www.interdb.jp/pg/pgsql09.html)  
[PostgreSQL 源码解读（4）-插入数据#3（heap_insert）](https://www.jianshu.com/p/0e16fce4ca73)  
[PostgreSQL 事务日志WAL结构浅析](https://www.jianshu.com/p/b9087d9f20e2)  
[PG Source Code](https://doxygen.postgresql.org)

