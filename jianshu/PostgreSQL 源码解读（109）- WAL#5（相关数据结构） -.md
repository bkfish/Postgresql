本节简单介绍了WAL相关的数据结构，包括XLogLongPageHeaderData、XLogPageHeaderData和XLogRecord。

### 一、数据结构

**XLogPageHeaderData**  
每一个事务日志文件(WAL segment file)的page(大小默认为8K)都有头部数据.  
注:每个文件第一个page的头部数据是XLogLongPageHeaderData(详见后续描述),而不是XLogPageHeaderData

    
    
    /*
     * Each page of XLOG file has a header like this:
     * 每一个事务日志文件的page都有头部信息,结构如下:
     */
    //可作为WAL版本信息
    #define XLOG_PAGE_MAGIC 0xD098  /* can be used as WAL version indicator */
    
    typedef struct XLogPageHeaderData
    {
        //WAL版本信息,PG V11.1 --> 0xD98
        uint16      xlp_magic;      /* magic value for correctness checks */
        //标记位(详见下面说明)
        uint16      xlp_info;       /* flag bits, see below */
        //page中第一个XLOG Record的TimeLineID,类型为uint32
        TimeLineID  xlp_tli;        /* TimeLineID of first record on page */
        //page的XLOG地址(在事务日志中的偏移),类型为uint64
        XLogRecPtr  xlp_pageaddr;   /* XLOG address of this page */
    
        /*
         * When there is not enough space on current page for whole record, we
         * continue on the next page.  xlp_rem_len is the number of bytes
         * remaining from a previous page.
         * 如果当前页的空间不足以存储整个XLOG Record,在下一个页面中存储余下的数据
         * xlp_rem_len表示上一页XLOG Record剩余部分的大小
         *
         * Note that xl_rem_len includes backup-block data; that is, it tracks
         * xl_tot_len not xl_len in the initial header.  Also note that the
         * continuation data isn't necessarily aligned.
         * 注意xl_rem_len包含backup-block data(full-page-write);
         * 也就是说在初始的头部信息中跟踪的是xl_tot_len而不是xl_len.
         * 另外要注意的是剩余的数据不需要对齐.
         */
        //上一页空间不够存储XLOG Record,该Record在本页继续存储占用的空间大小
        uint32      xlp_rem_len;    /* total len of remaining data for record */
    } XLogPageHeaderData;
    
    #define SizeOfXLogShortPHD  MAXALIGN(sizeof(XLogPageHeaderData))
    
    typedef XLogPageHeaderData *XLogPageHeader;
    
    

**XLogLongPageHeaderData**  
如设置了XLP_LONG_HEADER标记,在page header中存储额外的字段.  
(通常在每个事务日志文件也就是segment file的的第一个page中存在).  
这些附加的字段用于准确的识别文件。

    
    
    /*
     * When the XLP_LONG_HEADER flag is set, we store additional fields in the
     * page header.  (This is ordinarily done just in the first page of an
     * XLOG file.)  The additional fields serve to identify the file accurately.
     * 如设置了XLP_LONG_HEADER标记,在page header中存储额外的字段.
     * (通常在每个事务日志文件也就是segment file的的第一个page中存在).
     * 附加字段用于准确识别文件。
     */
    typedef struct XLogLongPageHeaderData
    {
        //标准的头部域字段
        XLogPageHeaderData std;     /* standard header fields */
        //pg_control中的系统标识码
        uint64      xlp_sysid;      /* system identifier from pg_control */
        //交叉检查
        uint32      xlp_seg_size;   /* just as a cross-check */
        //交叉检查
        uint32      xlp_xlog_blcksz;    /* just as a cross-check */
    } XLogLongPageHeaderData;
    
    #define SizeOfXLogLongPHD   MAXALIGN(sizeof(XLogLongPageHeaderData))
    //指针
    typedef XLogLongPageHeaderData *XLogLongPageHeader;
    
    /* When record crosses page boundary, set this flag in new page's header */
    //如果XLOG Record跨越page边界,在新page header中设置该标志位
    #define XLP_FIRST_IS_CONTRECORD     0x0001
    //该标志位标明是"long"页头
    /* This flag indicates a "long" page header */
    #define XLP_LONG_HEADER             0x0002
    /* This flag indicates backup blocks starting in this page are optional */
    //该标志位标明从该页起始的backup blocks是可选的(不一定存在)
    #define XLP_BKP_REMOVABLE           0x0004
    //xlp_info中所有定义的标志位(用于page header的有效性检查)
    /* All defined flag bits in xlp_info (used for validity checking of header) */
    #define XLP_ALL_FLAGS               0x0007
    
    #define XLogPageHeaderSize(hdr)     \
        (((hdr)->xlp_info & XLP_LONG_HEADER) ? SizeOfXLogLongPHD : SizeOfXLogShortPHD)
    
    

**XLogRecord**  
事务日志文件由N个的XLog Record组成,逻辑上对应XLOG Record这一概念的数据结构是XLogRecord.  
XLOG Record的整体布局如下:  
头部数据(固定大小的XLogRecord结构体)  
XLogRecordBlockHeader 结构体  
XLogRecordBlockHeader 结构体  
...  
XLogRecordDataHeader[Short|Long] 结构体  
block data  
block data  
...  
main data  
XLOG Record按存储的数据内容来划分,大体可以分为三类:  
1.Record for backup block:存储full-write-page的block,这种类型Record的目的是为了解决page部分写的问题;  
2.Record for (tuple)data block:在full-write-page后,相应的page中的tuple变更,使用这种类型的Record记录;  
3.Record for Checkpoint:在checkpoint发生时,在事务日志文件中记录checkpoint信息(其中包括Redo point).

_XLOG Record的详细解析后续会解析,这里暂且不提_

    
    
    /*
     * The overall layout of an XLOG record is:
     *      Fixed-size header (XLogRecord struct)
     *      XLogRecordBlockHeader struct
     *      XLogRecordBlockHeader struct
     *      ...
     *      XLogRecordDataHeader[Short|Long] struct
     *      block data
     *      block data
     *      ...
     *      main data
     * XLOG record的整体布局如下:
     *      固定大小的头部(XLogRecord 结构体)
     *      XLogRecordBlockHeader 结构体
     *      XLogRecordBlockHeader 结构体
     *      ...
     *      XLogRecordDataHeader[Short|Long] 结构体
     *      block data
     *      block data
     *      ...
     *      main data
     *
     * There can be zero or more XLogRecordBlockHeaders, and 0 or more bytes of
     * rmgr-specific data not associated with a block.  XLogRecord structs
     * always start on MAXALIGN boundaries in the WAL files, but the rest of
     * the fields are not aligned.
     * 其中,XLogRecordBlockHeaders可能有0或者多个,与block无关的0或多个字节的rmgr-specific数据
     * XLogRecord通常在WAL文件的MAXALIGN边界起写入,但后续的字段并没有对齐
     *
     * The XLogRecordBlockHeader, XLogRecordDataHeaderShort and
     * XLogRecordDataHeaderLong structs all begin with a single 'id' byte. It's
     * used to distinguish between block references, and the main data structs.
     * XLogRecordBlockHeader/XLogRecordDataHeaderShort/XLogRecordDataHeaderLong开头是占用1个字节的"id".
     * 用于区分block引用和main data结构体.
     */
    typedef struct XLogRecord
    {
        //record的大小
        uint32      xl_tot_len;     /* total len of entire record */
        //xact id
        TransactionId xl_xid;       /* xact id */
        //指向log中的前一条记录
        XLogRecPtr  xl_prev;        /* ptr to previous record in log */
        //标识位,详见下面的说明
        uint8       xl_info;        /* flag bits, see below */
        //该记录的资源管理器
        RmgrId      xl_rmid;        /* resource manager for this record */
        /* 2 bytes of padding here, initialize to zero */
        //2个字节的crc校验位,初始化为0
        pg_crc32c   xl_crc;         /* CRC for this record */
    
        /* XLogRecordBlockHeaders and XLogRecordDataHeader follow, no padding */
        //接下来是XLogRecordBlockHeaders和XLogRecordDataHeader
    } XLogRecord;
    //宏定义:XLogRecord大小
    #define SizeOfXLogRecord    (offsetof(XLogRecord, xl_crc) + sizeof(pg_crc32c))
    
    /*
     * The high 4 bits in xl_info may be used freely by rmgr. The
     * XLR_SPECIAL_REL_UPDATE and XLR_CHECK_CONSISTENCY bits can be passed by
     * XLogInsert caller. The rest are set internally by XLogInsert.
     * xl_info的高4位由rmgr自由使用.
     * XLR_SPECIAL_REL_UPDATE和XLR_CHECK_CONSISTENCY由XLogInsert函数的调用者传入.
     * 其余由XLogInsert内部使用.
     */
    #define XLR_INFO_MASK           0x0F
    #define XLR_RMGR_INFO_MASK      0xF0
    
    /*
     * If a WAL record modifies any relation files, in ways not covered by the
     * usual block references, this flag is set. This is not used for anything
     * by PostgreSQL itself, but it allows external tools that read WAL and keep
     * track of modified blocks to recognize such special record types.
     * 如果WAL记录使用特殊的方式(不涉及通常块引用)更新了关系的存储文件,设置此标记.
     * PostgreSQL本身并不使用这种方法，但它允许外部工具读取WAL并跟踪修改后的块，
     *   以识别这种特殊的记录类型。
     */
    #define XLR_SPECIAL_REL_UPDATE  0x01
    
    /*
     * Enforces consistency checks of replayed WAL at recovery. If enabled,
     * each record will log a full-page write for each block modified by the
     * record and will reuse it afterwards for consistency checks. The caller
     * of XLogInsert can use this value if necessary, but if
     * wal_consistency_checking is enabled for a rmgr this is set unconditionally.
     * 在恢复时强制执行一致性检查.
     * 如启用此功能,每个记录将为记录修改的每个块记录一个完整的页面写操作，并在以后重用它进行一致性检查。
     * 在需要时,XLogInsert的调用者可使用此标记,但如果rmgr启用了wal_consistency_checking,
     *   则会无条件执行一致性检查.
     */
    #define XLR_CHECK_CONSISTENCY   0x02
    
    
    /*
     * Header info for block data appended to an XLOG record.
     * 追加到XLOG record中block data的头部信息
     *
     * 'data_length' is the length of the rmgr-specific payload data associated
     * with this block. It does not include the possible full page image, nor
     * XLogRecordBlockHeader struct itself.
     * 'data_length'是与此块关联的rmgr特定payload data的长度。
     * 它不包括可能的full page image，也不包括XLogRecordBlockHeader结构体本身。
     *
     * Note that we don't attempt to align the XLogRecordBlockHeader struct!
     * So, the struct must be copied to aligned local storage before use.
     * 注意:我们不打算尝试对齐XLogRecordBlockHeader结构体!
     * 因此,在使用前,XLogRecordBlockHeader必须拷贝到一队齐的本地存储中.
     */
    typedef struct XLogRecordBlockHeader
    {
        //块引用ID
        uint8       id;             /* block reference ID */
        //在关系中使用的fork和flags
        uint8       fork_flags;     /* fork within the relation, and flags */
        //payload字节大小
        uint16      data_length;    /* number of payload bytes (not including page
                                     * image) */
    
        /* If BKPBLOCK_HAS_IMAGE, an XLogRecordBlockImageHeader struct follows */
        //如BKPBLOCK_HAS_IMAGE,后续为XLogRecordBlockImageHeader结构体
        /* If BKPBLOCK_SAME_REL is not set, a RelFileNode follows */
        //如BKPBLOCK_SAME_REL没有设置,则为RelFileNode
        /* BlockNumber follows */
        //后续为BlockNumber
    } XLogRecordBlockHeader;
     
    #define SizeOfXLogRecordBlockHeader (offsetof(XLogRecordBlockHeader, data_length) + sizeof(uint16))
    
    /*
     * Additional header information when a full-page image is included
     * (i.e. when BKPBLOCK_HAS_IMAGE is set).
     * 当包含完整页图像时(即当设置BKPBLOCK_HAS_IMAGE时)，附加的头部信息。
     *
     * The XLOG code is aware that PG data pages usually contain an unused "hole"
     * in the middle, which contains only zero bytes.  Since we know that the
     * "hole" is all zeros, we remove it from the stored data (and it's not counted
     * in the XLOG record's CRC, either).  Hence, the amount of block data actually
     * present is (BLCKSZ - <length of "hole" bytes>).
     * XLOG代码知道PG数据页通常在中间包含一个未使用的“hole”(空闲空间)，
     *   大小为零字节。
     * 因为我们知道“hole”都是零，
     *   以我们从存储的数据中删除它(而且它也没有被计入XLOG记录的CRC中)。
     * 因此，实际呈现的块数据量为(BLCKSZ - <“hole”的大小>)。
     *
     * Additionally, when wal_compression is enabled, we will try to compress full
     * page images using the PGLZ compression algorithm, after removing the "hole".
     * This can reduce the WAL volume, but at some extra cost of CPU spent
     * on the compression during WAL logging. In this case, since the "hole"
     * length cannot be calculated by subtracting the number of page image bytes
     * from BLCKSZ, basically it needs to be stored as an extra information.
     * But when no "hole" exists, we can assume that the "hole" length is zero
     * and no such an extra information needs to be stored. Note that
     * the original version of page image is stored in WAL instead of the
     * compressed one if the number of bytes saved by compression is less than
     * the length of extra information. Hence, when a page image is successfully
     * compressed, the amount of block data actually present is less than
     * BLCKSZ - the length of "hole" bytes - the length of extra information.
     * 另外，在启用wal_compression时，会在去掉“hole”后，尝试使用PGLZ压缩算法压缩full page image。
     * 这可以简化WAL大小,但会增加额外的解压缩CPU时间.
     * 在这种情况下，由于“hole”的长度不能通过从BLCKSZ中减去page image字节数来计算，
     *   所以它基本上需要作为额外的信息来存储。
     * 但如果"hole"不存在,我们可以假设"hole"的大小为0,不需要存储额外的信息.
     * 请注意，如果压缩节省的字节数小于额外信息的长度，
     *   那么page image的原始版本存储在WAL中，而不是压缩后的版本。
     * 因此，当一个page image被成功压缩时，
     *   实际的块数据量小于BLCKSZ - “hole”的大小 - 额外信息的大小。
     */
    typedef struct XLogRecordBlockImageHeader
    {
        uint16      length;         /* number of page image bytes */
        uint16      hole_offset;    /* number of bytes before "hole" */
        uint8       bimg_info;      /* flag bits, see below */
    
        /*
         * If BKPIMAGE_HAS_HOLE and BKPIMAGE_IS_COMPRESSED, an
         * XLogRecordBlockCompressHeader struct follows.
         * 如标记BKPIMAGE_HAS_HOLE和BKPIMAGE_IS_COMPRESSED设置,则后跟XLogRecordBlockCompressHeader
         */
    } XLogRecordBlockImageHeader;
    
    #define SizeOfXLogRecordBlockImageHeader    \
        (offsetof(XLogRecordBlockImageHeader, bimg_info) + sizeof(uint8))
    
    /* Information stored in bimg_info */
    //------------ bimg_info标记位
    //存在"hole"
    #define BKPIMAGE_HAS_HOLE       0x01    /* page image has "hole" */
    //压缩存储
    #define BKPIMAGE_IS_COMPRESSED      0x02    /* page image is compressed */
    //在回放时,page image需要恢复
    #define BKPIMAGE_APPLY      0x04    /* page image should be restored during
                                         * replay */
    
    /*
     * Extra header information used when page image has "hole" and
     * is compressed.
     * page image存在"hole"和压缩存储时,额外的头部信息
     */
    typedef struct XLogRecordBlockCompressHeader
    {
        //"hole"的大小
        uint16      hole_length;    /* number of bytes in "hole" */
    } XLogRecordBlockCompressHeader;
    
    #define SizeOfXLogRecordBlockCompressHeader \
        sizeof(XLogRecordBlockCompressHeader)
    
    /*
     * Maximum size of the header for a block reference. This is used to size a
     * temporary buffer for constructing the header.
     * 块引用的header的最大大小。
     * 它用于设置用于构造头部临时缓冲区的大小。
     */
    #define MaxSizeOfXLogRecordBlockHeader \
        (SizeOfXLogRecordBlockHeader + \
         SizeOfXLogRecordBlockImageHeader + \
         SizeOfXLogRecordBlockCompressHeader + \
         sizeof(RelFileNode) + \
         sizeof(BlockNumber))
    
    /*
     * The fork number fits in the lower 4 bits in the fork_flags field. The upper
     * bits are used for flags.
     * fork号适合于fork_flags字段的低4位。
     * 高4位用于标记。
     */
    #define BKPBLOCK_FORK_MASK  0x0F
    #define BKPBLOCK_FLAG_MASK  0xF0
    //块数据是XLogRecordBlockImage
    #define BKPBLOCK_HAS_IMAGE  0x10    /* block data is an XLogRecordBlockImage */
    #define BKPBLOCK_HAS_DATA   0x20
    //重做时重新初始化page
    #define BKPBLOCK_WILL_INIT  0x40    /* redo will re-init the page */
    //重做时重新初始化page,但会省略RelFileNode
    #define BKPBLOCK_SAME_REL   0x80    /* RelFileNode omitted, same as previous */
    
    /*
     * XLogRecordDataHeaderShort/Long are used for the "main data" portion of
     * the record. If the length of the data is less than 256 bytes, the short
     * form is used, with a single byte to hold the length. Otherwise the long
     * form is used.
     * XLogRecordDataHeaderShort/Long用于记录的“main data”部分。
     * 如果数据的长度小于256字节，则使用短格式，用一个字节保存长度。
     * 否则使用长形式。
     *
     * (These structs are currently not used in the code, they are here just for
     * documentation purposes).
     * (这些结构体不会再代码中使用,在这里是为了文档记录的目的)
     */
    typedef struct XLogRecordDataHeaderShort
    {
        uint8       id;             /* XLR_BLOCK_ID_DATA_SHORT */
        uint8       data_length;    /* number of payload bytes */
    }           XLogRecordDataHeaderShort;
    
    #define SizeOfXLogRecordDataHeaderShort (sizeof(uint8) * 2)
    
    typedef struct XLogRecordDataHeaderLong
    {
        uint8       id;             /* XLR_BLOCK_ID_DATA_LONG */
        /* followed by uint32 data_length, unaligned */
        //接下来是无符号32位整型的data_length(未对齐)
    }           XLogRecordDataHeaderLong;
    
    #define SizeOfXLogRecordDataHeaderLong (sizeof(uint8) + sizeof(uint32))
    
    /*
     * Block IDs used to distinguish different kinds of record fragments. Block
     * references are numbered from 0 to XLR_MAX_BLOCK_ID. A rmgr is free to use
     * any ID number in that range (although you should stick to small numbers,
     * because the WAL machinery is optimized for that case). A couple of ID
     * numbers are reserved to denote the "main" data portion of the record.
     * 块id用于区分不同类型的记录片段。
     * 块引用编号从0到XLR_MAX_BLOCK_ID。
     * rmgr可以自由使用该范围内的任何ID号
     *   (尽管您应该坚持使用较小的数字，因为WAL机制针对这种情况进行了优化)。
     * 保留两个ID号来表示记录的“main”数据部分。
     *
     * The maximum is currently set at 32, quite arbitrarily. Most records only
     * need a handful of block references, but there are a few exceptions that
     * need more.
     * 目前的最大值是32，非常随意。
     * 大多数记录只需要少数块引用，但也有少数例外需要更多。
     */
    #define XLR_MAX_BLOCK_ID            32
    
    #define XLR_BLOCK_ID_DATA_SHORT     255
    #define XLR_BLOCK_ID_DATA_LONG      254
    #define XLR_BLOCK_ID_ORIGIN         253
    
    #endif                          /* XLOGRECORD_H */
    
    

**这些数据结构在WAL segment file文件中如何布局,请参见后续的章节**

### 二、参考资料

[Write Ahead Logging — WAL](http://www.interdb.jp/pg/pgsql09.html)  
[PostgreSQL 源码解读（4）-插入数据#3（heap_insert）](https://www.jianshu.com/p/0e16fce4ca73)  
[PG Source Code](https://doxygen.postgresql.org)

