本文简单介绍了PG插入数据部分的源码，主要内容包括RelationPutHeapTuple函数的实现逻辑。

### 一、数据结构/宏定义/通用函数

RelationPutHeapTuple函数在hio.c文件中，相关的数据结构、宏定义如下：

    
    
    1、Relation
    数据表数据结构封装
     typedef struct RelationData
     {
         RelFileNode rd_node;        /* relation physical identifier */
         /* use "struct" here to avoid needing to include smgr.h: */
         struct SMgrRelationData *rd_smgr;   /* cached file handle, or NULL */
         int         rd_refcnt;      /* reference count */
         BackendId   rd_backend;     /* owning backend id, if temporary relation */
         bool        rd_islocaltemp; /* rel is a temp rel of this session */
         bool        rd_isnailed;    /* rel is nailed in cache */
         bool        rd_isvalid;     /* relcache entry is valid */
         char        rd_indexvalid;  /* state of rd_indexlist: 0 = not valid, 1 =
                                      * valid, 2 = temporarily forced */
         bool        rd_statvalid;   /* is rd_statlist valid? */
     
         /*
          * rd_createSubid is the ID of the highest subtransaction the rel has
          * survived into; or zero if the rel was not created in the current top
          * transaction.  This can be now be relied on, whereas previously it could
          * be "forgotten" in earlier releases. Likewise, rd_newRelfilenodeSubid is
          * the ID of the highest subtransaction the relfilenode change has
          * survived into, or zero if not changed in the current transaction (or we
          * have forgotten changing it). rd_newRelfilenodeSubid can be forgotten
          * when a relation has multiple new relfilenodes within a single
          * transaction, with one of them occurring in a subsequently aborted
          * subtransaction, e.g. BEGIN; TRUNCATE t; SAVEPOINT save; TRUNCATE t;
          * ROLLBACK TO save; -- rd_newRelfilenode is now forgotten
          */
         SubTransactionId rd_createSubid;    /* rel was created in current xact */
         SubTransactionId rd_newRelfilenodeSubid;    /* new relfilenode assigned in
                                                      * current xact */
     
         Form_pg_class rd_rel;       /* RELATION tuple */
         TupleDesc   rd_att;         /* tuple descriptor */
         Oid         rd_id;          /* relation's object id */
         LockInfoData rd_lockInfo;   /* lock mgr's info for locking relation */
         RuleLock   *rd_rules;       /* rewrite rules */
         MemoryContext rd_rulescxt;  /* private memory cxt for rd_rules, if any */
         TriggerDesc *trigdesc;      /* Trigger info, or NULL if rel has none */
         /* use "struct" here to avoid needing to include rowsecurity.h: */
         struct RowSecurityDesc *rd_rsdesc;  /* row security policies, or NULL */
     
         /* data managed by RelationGetFKeyList: */
         List       *rd_fkeylist;    /* list of ForeignKeyCacheInfo (see below) */
         bool        rd_fkeyvalid;   /* true if list has been computed */
     
         MemoryContext rd_partkeycxt;    /* private memory cxt for the below */
         struct PartitionKeyData *rd_partkey;    /* partition key, or NULL */
         MemoryContext rd_pdcxt;     /* private context for partdesc */
         struct PartitionDescData *rd_partdesc;  /* partitions, or NULL */
         List       *rd_partcheck;   /* partition CHECK quals */
     
         /* data managed by RelationGetIndexList: */
         List       *rd_indexlist;   /* list of OIDs of indexes on relation */
         Oid         rd_oidindex;    /* OID of unique index on OID, if any */
         Oid         rd_pkindex;     /* OID of primary key, if any */
         Oid         rd_replidindex; /* OID of replica identity index, if any */
     
         /* data managed by RelationGetStatExtList: */
         List       *rd_statlist;    /* list of OIDs of extended stats */
     
         /* data managed by RelationGetIndexAttrBitmap: */
         Bitmapset  *rd_indexattr;   /* columns used in non-projection indexes */
         Bitmapset  *rd_projindexattr;   /* columns used in projection indexes */
         Bitmapset  *rd_keyattr;     /* cols that can be ref'd by foreign keys */
         Bitmapset  *rd_pkattr;      /* cols included in primary key */
         Bitmapset  *rd_idattr;      /* included in replica identity index */
         Bitmapset  *rd_projidx;     /* Oids of projection indexes */
     
         PublicationActions *rd_pubactions;  /* publication actions */
     
         /*
          * rd_options is set whenever rd_rel is loaded into the relcache entry.
          * Note that you can NOT look into rd_rel for this data.  NULL means "use
          * defaults".
          */
         bytea      *rd_options;     /* parsed pg_class.reloptions */
     
         /* These are non-NULL only for an index relation: */
         Form_pg_index rd_index;     /* pg_index tuple describing this index */
         /* use "struct" here to avoid needing to include htup.h: */
         struct HeapTupleData *rd_indextuple;    /* all of pg_index tuple */
     
         /*
          * index access support info (used only for an index relation)
          *
          * Note: only default support procs for each opclass are cached, namely
          * those with lefttype and righttype equal to the opclass's opcintype. The
          * arrays are indexed by support function number, which is a sufficient
          * identifier given that restriction.
          *
          * Note: rd_amcache is available for index AMs to cache private data about
          * an index.  This must be just a cache since it may get reset at any time
          * (in particular, it will get reset by a relcache inval message for the
          * index).  If used, it must point to a single memory chunk palloc'd in
          * rd_indexcxt.  A relcache reset will include freeing that chunk and
          * setting rd_amcache = NULL.
          */
         Oid         rd_amhandler;   /* OID of index AM's handler function */
         MemoryContext rd_indexcxt;  /* private memory cxt for this stuff */
         /* use "struct" here to avoid needing to include amapi.h: */
         struct IndexAmRoutine *rd_amroutine;    /* index AM's API struct */
         Oid        *rd_opfamily;    /* OIDs of op families for each index col */
         Oid        *rd_opcintype;   /* OIDs of opclass declared input data types */
         RegProcedure *rd_support;   /* OIDs of support procedures */
         FmgrInfo   *rd_supportinfo; /* lookup info for support procedures */
         int16      *rd_indoption;   /* per-column AM-specific flags */
         List       *rd_indexprs;    /* index expression trees, if any */
         List       *rd_indpred;     /* index predicate tree, if any */
         Oid        *rd_exclops;     /* OIDs of exclusion operators, if any */
         Oid        *rd_exclprocs;   /* OIDs of exclusion ops' procs, if any */
         uint16     *rd_exclstrats;  /* exclusion ops' strategy numbers, if any */
         void       *rd_amcache;     /* available for use by index AM */
         Oid        *rd_indcollation;    /* OIDs of index collations */
     
         /*
          * foreign-table support
          *
          * rd_fdwroutine must point to a single memory chunk palloc'd in
          * CacheMemoryContext.  It will be freed and reset to NULL on a relcache
          * reset.
          */
     
         /* use "struct" here to avoid needing to include fdwapi.h: */
         struct FdwRoutine *rd_fdwroutine;   /* cached function pointers, or NULL */
     
         /*
          * Hack for CLUSTER, rewriting ALTER TABLE, etc: when writing a new
          * version of a table, we need to make any toast pointers inserted into it
          * have the existing toast table's OID, not the OID of the transient toast
          * table.  If rd_toastoid isn't InvalidOid, it is the OID to place in
          * toast pointers inserted into this rel.  (Note it's set on the new
          * version of the main heap, not the toast table itself.)  This also
          * causes toast_save_datum() to try to preserve toast value OIDs.
          */
         Oid         rd_toastoid;    /* Real TOAST table's OID, or InvalidOid */
     
         /* use "struct" here to avoid needing to include pgstat.h: */
         struct PgStat_TableStatus *pgstat_info; /* statistics collection area */
     } RelationData;
     
    typedef struct RelationData *Relation;
    2、Buffer
    实际类型为整型，共享缓冲区的index，0为非法Buffer。
     /*
      * Buffer identifiers.
      *
      * Zero is invalid, positive is the index of a shared buffer (1..NBuffers),
      * negative is the index of a local buffer (-1 .. -NLocBuffer).
      */
     typedef int Buffer;
     
     #define InvalidBuffer   0
    
    3、HeapTupleHeader
    Heap（还有一种是Index）类型Tuple的头部数据，在Page结构中已作详细分析。
     struct HeapTupleHeaderData
     {
         union
         {
             HeapTupleFields t_heap;
             DatumTupleFields t_datum;
         }           t_choice;
          ItemPointerData t_ctid;     /* current TID of this or newer tuple (or a
                                      * speculative insertion token) */
         /* Fields below here must match MinimalTupleData! */
      #define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK2 2
         uint16      t_infomask2;    /* number of attributes + various flags */
      #define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK 3
         uint16      t_infomask;     /* various flag bits, see below */
      #define FIELDNO_HEAPTUPLEHEADERDATA_HOFF 4
         uint8       t_hoff;         /* sizeof header incl. bitmap, padding */
          /* ^ - 23 bytes - ^ */
      #define FIELDNO_HEAPTUPLEHEADERDATA_BITS 5
         bits8       t_bits[FLEXIBLE_ARRAY_MEMBER];  /* bitmap of NULLs */
          /* MORE DATA FOLLOWS AT END OF STRUCT */
     };
    
    4、ItemPointerData
    数据行指针数据结构，ip_blkid是数据块ID，ip_posid是Tuple在数据块中的偏移（其实是类似数组中的序号）。
    typedef struct ItemPointerData
     {
         BlockIdData ip_blkid;
         OffsetNumber ip_posid;
     }  ItemPointerData;
     
     typedef ItemPointerData *ItemPointer;
    
     typedef struct BlockIdData
     {
         uint16      bi_hi;
         uint16      bi_lo;
     } BlockIdData;
     
     typedef BlockIdData *BlockId; /* block identifier */
    
    5、HeapTuple
    存储在Heap中的Tuple（Row）数据结构：
    
    typedef struct HeapTupleData
     {
         uint32      t_len;          /* length of *t_data */
         ItemPointerData t_self;     /* SelfItemPointer */
         Oid         t_tableOid;     /* table the tuple came from */
     #define FIELDNO_HEAPTUPLEDATA_DATA 3
         HeapTupleHeader t_data;     /* -> tuple header and data */
     } HeapTupleData;
     
     typedef HeapTupleData *HeapTuple;
     
     #define HEAPTUPLESIZE   MAXALIGN(sizeof(HeapTupleData))
    
    6、HeapTupleHeaderIsSpeculative
     #define HeapTupleHeaderIsSpeculative(tup) \
     ( \
      (ItemPointerGetOffsetNumberNoCheck(&(tup)->t_ctid) == SpecTokenOffsetNumber) \
     )
    
     #define ItemPointerGetOffsetNumberNoCheck(pointer) \
     ( \
      (pointer)->ip_posid \
     )
    
    7、BufferGetPage
    //获取与该buffer（有符号整型）对应的page
     #define BufferGetPage(buffer) ((Page)BufferGetBlock(buffer))
     #define BufferGetBlock(buffer) \
     ( \
      AssertMacro(BufferIsValid(buffer)), \
      BufferIsLocal(buffer) ? \
      LocalBufferBlockPointers[-(buffer) - 1] \
      : \
      (Block) (BufferBlocks + ((Size) ((buffer) - 1)) * BLCKSZ) \
     )
     #define BufferIsLocal(buffer) ((buffer) < 0)
     typedef void *Block;//指向任意类型的指针
     Block *LocalBufferBlockPointers = NULL;//指针的指针
    
    8、BufferGetBlockNumber
     /*
      * BufferGetBlockNumber
      *      Returns the block number associated with a buffer.
      *
      * Note:
      *      Assumes that the buffer is valid and pinned, else the
      *      value may be obsolete immediately...
      */
     BlockNumber
     BufferGetBlockNumber(Buffer buffer)
     {
         BufferDesc *bufHdr;
     
         Assert(BufferIsPinned(buffer));
     
         if (BufferIsLocal(buffer))
             bufHdr = GetLocalBufferDescriptor(-buffer - 1);
         else
             bufHdr = GetBufferDescriptor(buffer - 1);
     
         /* pinned, so OK to read tag without spinlock */
         return bufHdr->tag.blockNum;
     }
    
    9、BlockIdSet
     /*
      * BlockIdSet
      *      Sets a block identifier to the specified value.
      */
     #define BlockIdSet(blockId, blockNumber) \
     ( \
         AssertMacro(PointerIsValid(blockId)), \
         (blockId)->bi_hi = (blockNumber) >> 16, \//右移16位，得到高位
         (blockId)->bi_lo = (blockNumber) & 0xffff \//高16位全部置0，得到低位
     )
    
    10、ItemPointerSet
     /*
      * ItemPointerSet
      * Sets a disk item pointer to the specified block and offset.
      */
     #define ItemPointerSet(pointer, blockNumber, offNum) \
     ( \
      AssertMacro(PointerIsValid(pointer)), \
      BlockIdSet(&((pointer)->ip_blkid), blockNumber), \
      (pointer)->ip_posid = offNum \
     )
    
    11、PageGetItemId
    获取行指针（ItemIdData指针） 
    /*
      * PageGetItemId
      * Returns an item identifier of a page.
      */
     #define PageGetItemId(page, offsetNumber) \
      ((ItemId) (&((PageHeader) (page))->pd_linp[(offsetNumber) - 1]))
    
    12、PageGetItem
    根据ItemId获取相应的Item（Tuple）
     /*
      * PageGetItem
      *      Retrieves an item on the given page.
      *
      * Note:
      *      This does not change the status of any of the resources passed.
      *      The semantics may change in the future.
      */
     #define PageGetItem(page, itemId) \
     ( \
         AssertMacro(PageIsValid(page)), \
         AssertMacro(ItemIdHasStorage(itemId)), \
         (Item)(((char *)(page)) + ItemIdGetOffset(itemId)) \
     )
    
     #define ItemIdGetOffset(itemId) \
      ((itemId)->lp_off)
    
    

### 二、源码解读

    
    
    /*
     * RelationPutHeapTuple - place tuple at specified page
     *
     * !!! EREPORT(ERROR) IS DISALLOWED HERE !!!  Must PANIC on failure!!!
     *
     * Note - caller must hold BUFFER_LOCK_EXCLUSIVE on the buffer.
     */
    void
    RelationPutHeapTuple(Relation relation,
                         Buffer buffer,
                         HeapTuple tuple,
                         bool token)
    {
        Page        pageHeader;//页头
        OffsetNumber offnum;//行偏移
    
        /*
         * A tuple that's being inserted speculatively should already have its
         * token set.
         */
        //TODO token & speculatively有待考究
        Assert(!token || HeapTupleHeaderIsSpeculative(tuple->t_data));
    
        /* Add the tuple to the page */
        //根据buffer获取相应的page（页头）
        pageHeader = BufferGetPage(buffer);
        //插入数据,PageAddItem函数上一节已介绍，函数成功返回行偏移
       /*
       输入：
          page-指向Page的指针
          item-指向数据的指针
          size-数据大小
          offsetNumber-数据存储的偏移量，InvalidOffsetNumber表示不指定
          flags-不"覆盖"原数据
          is_heap-Heap数据
        输出：
          OffsetNumber-数据存储实际的偏移量
        */
        offnum = PageAddItem(pageHeader, (Item) tuple->t_data,
                             tuple->t_len, InvalidOffsetNumber, false, true);
        //如果不成功，记录日志
        if (offnum == InvalidOffsetNumber)
            elog(PANIC, "failed to add tuple to page");
        
        /* Update tuple->t_self to the actual position where it was stored */
        //&(tuple->t_self)类型为ItemPointer，亦即行指针（ItemPointerData结构体指针）
        //根据buffer获取块号，把块号和行偏移写入行指针中
        ItemPointerSet(&(tuple->t_self), BufferGetBlockNumber(buffer), offnum);
    
        /*
         * Insert the correct position into CTID of the stored tuple, too (unless
         * this is a speculative insertion, in which case the token is held in
         * CTID field instead)
         */
        if (!token)
        {
            //获取行指针，ItemId即ItemIdData指针
            ItemId      itemId = PageGetItemId(pageHeader, offnum);
            //获取TupleHeader
            HeapTupleHeader item = (HeapTupleHeader) PageGetItem(pageHeader, itemId);
            //更新TupleHeader中的行指针
            item->t_ctid = tuple->t_self;
        }
    }
    

### 三、跟踪分析

使用上一节的数据表，回收垃圾后，插入一条记录。

    
    
    testdb=# vacuum t_insert;
    VACUUM
    testdb=# 
    testdb=# checkpoint;
    CHECKPOINT
    testdb=#  select pg_backend_pid();
     pg_backend_pid 
    ----------------               1582
    (1 row)
    

使用gdb进行跟踪分析：

    
    
    [root@localhost ~]# gdb -p 1582
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    ...
    (gdb) 
    

插入一条记录：

    
    
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(10,'10','10','10');
    (挂起）
    

回到gdb：

    
    
    (gdb) b RelationPutHeapTuple
    Breakpoint 1 at 0x4cf492: file hio.c, line 51.
    #查看输入参数
    (gdb) p *relation
    $5 = {rd_node = {spcNode = 1663, dbNode = 16477, relNode = 26731}, rd_smgr = 0x259db68, rd_refcnt = 1, rd_backend = -1, rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, 
      rd_indexvalid = 0 '\000', rd_statvalid = false, rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7fa9814589e8, rd_att = 0x7fa981458af8, rd_id = 26731, rd_lockInfo = {lockRelId = {
          relId = 26731, dbId = 16477}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, rd_rsdesc = 0x0, rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, 
      rd_partdesc = 0x0, rd_partcheck = 0x0, rd_indexlist = 0x0, rd_oidindex = 0, rd_pkindex = 0, rd_replidindex = 0, rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, rd_keyattr = 0x0, 
      rd_pkattr = 0x0, rd_idattr = 0x0, rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, rd_indextuple = 0x0, rd_amhandler = 0, rd_indexcxt = 0x0, rd_amroutine = 0x0, 
      rd_opfamily = 0x0, rd_opcintype = 0x0, rd_support = 0x0, rd_supportinfo = 0x0, rd_indoption = 0x0, rd_indexprs = 0x0, rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, rd_exclstrats = 0x0, 
      rd_amcache = 0x0, rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, pgstat_info = 0x2591850}
    (gdb) p buffer
    $6 = 95
    (gdb) p tuple
    $7 = (HeapTuple) 0x2539a20
    (gdb) p *tuple  #注：HeapTuple
    $8 = {t_len = 61, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_tableOid = 26731, t_data = 0x2539a38}
    (gdb) p *tuple->t_data #注：HeapTupleHeader
    $9 = {t_choice = {t_heap = {t_xmin = 1612851, t_xmax = 0, t_field3 = {t_cid = 0, t_xvac = 0}}, t_datum = {datum_len_ = 1612851, datum_typmod = 0, datum_typeid = 0}}, t_ctid = {ip_blkid = {
          bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_infomask2 = 4, t_infomask = 2050, t_hoff = 24 '\030', t_bits = 0x2539a4f ""}
    (gdb) p token
    $10 = false
    #查看PageHeader信息
    (gdb) p *(PageHeader)pageHeader
    $11 = {pd_lsn = {xlogid = 1, xrecoff = 3677464616}, pd_checksum = 0, pd_flags = 5, pd_lower = 60, pd_upper = 7680, pd_special = 8192, pd_pagesize_version = 8196, pd_prune_xid = 0, 
      pd_linp = 0x7fa96957d318}
    #调用PageAddItem函数后
    (gdb) next
    56      if (offnum == InvalidOffsetNumber)
    (gdb) p offnum #2号Item被删除，在执行vacuum回收后，已可用
    $12 = 2
    (gdb) p *itemId
    $13 = {lp_off = 7616, lp_flags = 1, lp_len = 61}
    (gdb) p *item
    $14 = {t_choice = {t_heap = {t_xmin = 1612851, t_xmax = 0, t_field3 = {t_cid = 0, t_xvac = 0}}, t_datum = {datum_len_ = 1612851, datum_typmod = 0, datum_typeid = 0}}, t_ctid = {ip_blkid = {
          bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_infomask2 = 4, t_infomask = 2050, t_hoff = 24 '\030', t_bits = 0x7fa96957f0d7 ""}
    (gdb) next
    74  }
    (gdb) p *item
    No symbol "item" in current context.
    (gdb) p tuple->t_self
    $15 = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 2} #0号Block，2号偏移
    (gdb) c
    Continuing.
    

可以看到，这行数据“正确”的插入在0号Block，2号偏移的位置上。

### 四、小结

1、基本理解RelationPutHeapTuple函数的实现逻辑和相关的数据结构；  
2、在熟悉数据结构（包括宏定义&通用函数）的基础上，阅读源代码和使用gdb调试可以深入掌握PG处理数据“背后”的逻辑。  
下一节，将会讲述调用栈中heap_insert函数。

