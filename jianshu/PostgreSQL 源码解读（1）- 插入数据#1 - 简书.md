本文简单介绍了PG插入数据部分的源码，主要是PageAddItemExtended函数的逻辑，同时结合先前介绍的页存储结构通过gdb进行跟踪分析其中的数据结构。

### 一、测试数据表

testdb=# drop table if exists t_insert;  
NOTICE: table "t_insert" does not exist, skipping  
DROP TABLE  
testdb=# create table t_insert(id int,c1 char(10),c2 char(10),c3 char(10));  
CREATE TABLE

### 二、源码分析

插入数据主要的实现在bufpage.c中，主要的函数是PageAddItemExtended。该函数的英文注释已经说得非常清楚，不过为了更好理解，这里添加了中文注释。  
**变量、宏定义和结构体等说明**

    
    
    1、Page
    指向char的指针：
    typedef char *Pointer;
    typedef Pointer Page;
    2、Item
    指向char的指针：
    typedef Pointer Item;
    3、Size
    typedef size_t  Size;
    4、OffsetNumber
    unsigned short（16bits）：
    typedef uint16 OffsetNumber;
    5、PageHeader
    PageHeaderData结构体指针
    typedef struct PageHeaderData
     {
         /* XXX LSN is member of *any* block, not only page-organized ones */
         PageXLogRecPtr pd_lsn;      /* LSN: next byte after last byte of xlog
                                      * record for last change to this page */
         uint16      pd_checksum;    /* checksum */
         uint16      pd_flags;       /* flag bits, see below */
         LocationIndex pd_lower;     /* offset to start of free space */
         LocationIndex pd_upper;     /* offset to end of free space */
         LocationIndex pd_special;   /* offset to start of special space */
         uint16      pd_pagesize_version;
         TransactionId pd_prune_xid; /* oldest prunable XID, or zero if none */
         ItemIdData  pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
     } PageHeaderData;
     
     typedef PageHeaderData *PageHeader;
    
    #define PD_HAS_FREE_LINES   0x0001  /* are there any unused line pointers? */
    #define PD_PAGE_FULL        0x0002  /* not enough free space for new tuple? */
    #define PD_ALL_VISIBLE      0x0004  /* all tuples on page are visible to
                                         * everyone */
    #define PD_VALID_FLAG_BITS  0x0007  /* OR of all valid pd_flags bits */
    
    6、SizeOfPageHeaderData
    长整型，定义页头大小：
    #define offsetof(type, field)   ((long) &((type *)0)->field)
     #define SizeOfPageHeaderData (offsetof(PageHeaderData, pd_linp))
    7、BLCKSZ
    #define BLCKSZ 8192
    8、PageGetMaxOffsetNumber
    如果空闲空间的lower小于等于页头大小，值为0，否则值为lower减去页头大小（24Bytes）后除以ItemId大小（4Bytes）
     #define PageGetMaxOffsetNumber(page) \
         (((PageHeader) (page))->pd_lower <= SizeOfPageHeaderData ? 0 : \
          ((((PageHeader) (page))->pd_lower - SizeOfPageHeaderData) \
           / sizeof(ItemIdData)))
    9、InvalidOffsetNumber
    是否无效的偏移值
     #define InvalidOffsetNumber     ((OffsetNumber) 0)
    10、OffsetNumberNext
    输入值+1
     #define OffsetNumberNext(offsetNumber) \
         ((OffsetNumber) (1 + (offsetNumber)))
    11、ItemId
    结构体ItemIdData指针
     typedef struct ItemIdData
     {
         unsigned    lp_off:15,      /* offset to tuple (from start of page) */
                     lp_flags:2,     /* state of item pointer, see below */
                     lp_len:15;      /* byte length of tuple */
     } ItemIdData;
     
     typedef ItemIdData *ItemId;
    
     #define LP_UNUSED 0 /* unused (should always have lp_len=0) */
     #define LP_NORMAL 1 /* used (should always have lp_len>0) */
     #define LP_REDIRECT 2 /* HOT redirect (should have lp_len=0) */
     #define LP_DEAD 3 /* dead, may or may not have storage */
    
    12、PageGetItemId
    获取相应的ItemIdData的指针
    #define PageGetItemId(page, offsetNumber) \
        ((ItemId) (&((PageHeader) (page))->pd_linp[(offsetNumber) - 1]))
    13、PAI_OVERWRITE
    位标记，实际值为1
    #define PAI_OVERWRITE           (1 << 0)
    14、PAI_IS_HEAP
    位标记，实际值为2
    #define PAI_IS_HEAP             (1 << 1)
    15、ItemIdIsUsed
    ItemId是否已被占用
     #define ItemIdIsUsed(itemId) \
         ((itemId)->lp_flags != LP_UNUSED)
    16、ItemIdHasStorage
     #define ItemIdHasStorage(itemId) \
         ((itemId)->lp_len != 0
    17、PageHasFreeLinePointers/PageClearHasFreeLinePointers
    判断Page从Header到Lower之间是否有空位
    #define PageHasFreeLinePointers(page) \
        (((PageHeader) (page))->pd_flags & PD_HAS_FREE_LINES)
    清除空位标记（标记错误时清除）
    #define PageClearHasFreeLinePointers(page) \
        (((PageHeader) (page))->pd_flags &= ~PD_HAS_FREE_LINES)
    18、MaxHeapTuplesPerPage
    每个Page可以容纳的最大Tuple数，计算公式是：
    （Block大小 - 页头大小） / (对齐后行头部大小 + 行指针大小）
    #define MaxHeapTuplesPerPage    \
         ((int) ((BLCKSZ - SizeOfPageHeaderData) / \
                 (MAXALIGN(SizeofHeapTupleHeader) + sizeof(ItemIdData))))
    

**PageAddItemExtended函数解读**

    
    
    /*
    PageAddItemExtended函数：
    输入：
      page-指向页的指针
      item-指向数据的指针
      size-数据大小
      offsetNumber-指定数据存储的偏移量
      flags-标记位（是否覆盖/是否Heap数据）
    输出：
      OffsetNumber-数据存储实际的偏移量
    */
    OffsetNumber
    PageAddItemExtended(Page page,
                        Item item,
                        Size size,
                        OffsetNumber offsetNumber,
                        int flags)
    {
        PageHeader  phdr = (PageHeader) page;//页头指针
        Size        alignedSize;//对齐大小
        int         lower;//Free space低位
        int         upper;//Free space高位
        ItemId      itemId;//行指针
        OffsetNumber limit;//行偏移，Free space中第1个可用的位置偏移
        bool        needshuffle = false;//是否需要移动原有数据
    
        /*
         * Be wary about corrupted page pointers
         */
        if (phdr->pd_lower < SizeOfPageHeaderData ||
            phdr->pd_lower > phdr->pd_upper ||
            phdr->pd_upper > phdr->pd_special ||
            phdr->pd_special > BLCKSZ)
            ereport(PANIC,
                    (errcode(ERRCODE_DATA_CORRUPTED),
                     errmsg("corrupted page pointers: lower = %u, upper = %u, special = %u",
                            phdr->pd_lower, phdr->pd_upper, phdr->pd_special)));
    
        /*
         * Select offsetNumber to place the new item at
         */
            //获取存储数据的偏移（位于Lower和Upper之间）
        limit = OffsetNumberNext(PageGetMaxOffsetNumber(page));
    
        /* was offsetNumber passed in? */
        if (OffsetNumberIsValid(offsetNumber))
        {
            //如果指定了数据存储的偏移量（传入的偏移量参数有效）
            /* yes, check it */
            if ((flags & PAI_OVERWRITE) != 0) //不覆盖原有数据
            {
                if (offsetNumber < limit)
                {
                    //获取指定偏移的ItemId
                    itemId = PageGetItemId(phdr, offsetNumber);
                    //指定的数据偏移已使用或者已分配存储空间，报错
                    if (ItemIdIsUsed(itemId) || ItemIdHasStorage(itemId))
                    {
                        elog(WARNING, "will not overwrite a used ItemId");
                        return InvalidOffsetNumber;
                    }
                }
            }
            else//覆盖原有数据
            {
                //指定的行偏移不在空闲空间中，需要移动原数据为新数据腾位置
                if (offsetNumber < limit)
                    needshuffle = true; /* need to move existing linp's */
            }
        }
        else//没有指定数据存储行偏移
        {
            /* offsetNumber was not passed in, so find a free slot */
            /* if no free slot, we'll put it at limit (1st open slot) */
            if (PageHasFreeLinePointers(phdr))//页头标记提示存在已回收的空间
            {
                /*
                 * Look for "recyclable" (unused) ItemId.  We check for no storage
                 * as well, just to be paranoid --- unused items should never have
                 * storage.
                 */
                //循环找出第1个可用的空闲行偏移
                for (offsetNumber = 1; offsetNumber < limit; offsetNumber++)
                {
                    itemId = PageGetItemId(phdr, offsetNumber);
                    if (!ItemIdIsUsed(itemId) && !ItemIdHasStorage(itemId))
                        break;
                }
                //没有找到，说明页头标记有误，需清除标记，以免误导
                if (offsetNumber >= limit)
                {
                    /* the hint is wrong, so reset it */
                    PageClearHasFreeLinePointers(phdr);
                }
            }
            else//没有已回收的空间，行指针/数据存储到Free Space中
            {
                /* don't bother searching if hint says there's no free slot */
                offsetNumber = limit;
            }
        }
    
        /* Reject placing items beyond the first unused line pointer */
        if (offsetNumber > limit)
        {
            //如果指定的偏移大于空闲空间可用的第1个位置，报错
            elog(WARNING, "specified item offset is too large");
            return InvalidOffsetNumber;
        }
    
        /* Reject placing items beyond heap boundary, if heap */
        if ((flags & PAI_IS_HEAP) != 0 && offsetNumber > MaxHeapTuplesPerPage)
        {
            //Heap数据，但偏移大于一页可存储的最大Tuple数，报错
            elog(WARNING, "can't put more than MaxHeapTuplesPerPage items in a heap page");
            return InvalidOffsetNumber;
        }
    
        /*
         * Compute new lower and upper pointers for page, see if it'll fit.
         *
         * Note: do arithmetic as signed ints, to avoid mistakes if, say,
         * alignedSize > pd_upper.
         */
        if (offsetNumber == limit || needshuffle)
            //如果数据存储在Free space中，修改lower值
            lower = phdr->pd_lower + sizeof(ItemIdData);
        else
            lower = phdr->pd_lower;//否则，找到了已回收的空闲位置，使用原有的lower
    
        alignedSize = MAXALIGN(size);//大小对齐
    
        upper = (int) phdr->pd_upper - (int) alignedSize;//申请存储空间
    
        if (lower > upper)//校验
            return InvalidOffsetNumber;
    
        /*
         * OK to insert the item.  First, shuffle the existing pointers if needed.
         */
        //获取行指针
        itemId = PageGetItemId(phdr, offsetNumber);
        //
        if (needshuffle)
            //如果需要腾位置，把原有的行指针往后挪一"格"
            memmove(itemId + 1, itemId,
                    (limit - offsetNumber) * sizeof(ItemIdData));
    
        /* set the item pointer */
        //设置新数据行指针
        ItemIdSetNormal(itemId, upper, size);
    
        /*
         * Items normally contain no uninitialized bytes.  Core bufpage consumers
         * conform, but this is not a necessary coding rule; a new index AM could
         * opt to depart from it.  However, data type input functions and other
         * C-language functions that synthesize datums should initialize all
         * bytes; datumIsEqual() relies on this.  Testing here, along with the
         * similar check in printtup(), helps to catch such mistakes.
         *
         * Values of the "name" type retrieved via index-only scans may contain
         * uninitialized bytes; see comment in btrescan().  Valgrind will report
         * this as an error, but it is safe to ignore.
         */
        VALGRIND_CHECK_MEM_IS_DEFINED(item, size);
    
        /* copy the item's data onto the page */
        //把数据放在数据区
        memcpy((char *) page + upper, item, size);
    
        /* adjust page header */
        //更新页头的lower & upper
        phdr->pd_lower = (LocationIndex) lower;
        phdr->pd_upper = (LocationIndex) upper;
        //如成功，返回实际的行偏移
        return offsetNumber;
    }
    

### 三、跟踪分析

下面使用gdb对PageAddItemExtended函数跟踪分析。  
测试场景：首先插入8行数据，然后删除第2行，接着插入1行数据：

    
    
    testdb=# -- 插入8行数据
    testdb=# insert into t_insert values(1,'11','12','13');
    insert into t_insert values(6,'11','12','13');
    insert into t_insert values(7,'11','12','13');
    insert into t_insert values(8,'11','12','13');
    
    checkpoint;INSERT 0 1
    testdb=# insert into t_insert values(2,'11','12','13');
    INSERT 0 1
    testdb=# insert into t_insert values(3,'11','12','13');
    INSERT 0 1
    testdb=# insert into t_insert values(4,'11','12','13');
    INSERT 0 1
    testdb=# insert into t_insert values(5,'11','12','13');
    INSERT 0 1
    testdb=# insert into t_insert values(6,'11','12','13');
    INSERT 0 1
    testdb=# insert into t_insert values(7,'11','12','13');
    INSERT 0 1
    testdb=# insert into t_insert values(8,'11','12','13');
    INSERT 0 1
    testdb=# 
    testdb=# checkpoint;
    CHECKPOINT
    testdb=# -- 删除第2行
    testdb=# delete from t_insert where id = 2;
    DELETE 1
    testdb=# 
    testdb=# checkpoint;
    CHECKPOINT
    testdb=# 
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               1572
    (1 row)
    

使用gdb跟踪最后插入1行数据的过程：

    
    
    #启动gdb，绑定进程
    [root@localhost demo]# gdb -p 1572
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
    and "show warranty" for details.
    This GDB was configured as "x86_64-redhat-linux-gnu".
    For bug reporting instructions, please see:
    <http://www.gnu.org/software/gdb/bugs/>.
    Attaching to process 1572
    ...
    (gdb) 
    #设置断点
    (gdb) b PageAddItemExtended
    Breakpoint 1 at 0x845119: file bufpage.c, line 196.
    (gdb) 
    
    #切换到psql，执行插入语句
    testdb=# insert into t_insert values(9,'11','12','13');
    （挂起）
    
    #切换回gdb
    (gdb) c
    Continuing.
    
    Breakpoint 1, PageAddItemExtended (page=0x7feaaefac300 "\001", item=0x29859f8 "2\234\030", size=61, offsetNumber=0, flags=2) at bufpage.c:196
    196     PageHeader  phdr = (PageHeader) page;
    #断点在函数的第一行代码上
    #bt打印调用栈
    (gdb) bt
    #0  PageAddItemExtended (page=0x7feaaefac300 "\001", item=0x29859f8 "2\234\030", size=61, offsetNumber=0, flags=2) at bufpage.c:196
    #1  0x00000000004cf4f9 in RelationPutHeapTuple (relation=0x7feac6e2ccb8, buffer=141, tuple=0x29859e0, token=false) at hio.c:53
    #2  0x00000000004c34ec in heap_insert (relation=0x7feac6e2ccb8, tup=0x29859e0, cid=0, options=0, bistate=0x0) at heapam.c:2487
    #3  0x00000000006c076b in ExecInsert (mtstate=0x2984c10, slot=0x2985250, planSlot=0x2985250, estate=0x29848c0, canSetTag=true) at nodeModifyTable.c:529
    #4  0x00000000006c29f3 in ExecModifyTable (pstate=0x2984c10) at nodeModifyTable.c:2126
    #5  0x000000000069a7d8 in ExecProcNodeFirst (node=0x2984c10) at execProcnode.c:445
    #6  0x0000000000690994 in ExecProcNode (node=0x2984c10) at ../../../src/include/executor/executor.h:237
    #7  0x0000000000692e5e in ExecutePlan (estate=0x29848c0, planstate=0x2984c10, use_parallel_mode=false, operation=CMD_INSERT, sendTuples=false, numberTuples=0, direction=ForwardScanDirection, 
        dest=0x2990dc8, execute_once=true) at execMain.c:1726
    #8  0x0000000000690e58 in standard_ExecutorRun (queryDesc=0x2981020, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:363
    #9  0x0000000000690cef in ExecutorRun (queryDesc=0x2981020, direction=ForwardScanDirection, count=0, execute_once=true) at execMain.c:306
    #10 0x0000000000851d84 in ProcessQuery (plan=0x2990c68, sourceText=0x28c5ef0 "insert into t_insert values(9,'11','12','13');", params=0x0, queryEnv=0x0, dest=0x2990dc8, completionTag=0x7ffdbc052d10 "")
        at pquery.c:161
    #11 0x00000000008534f4 in PortalRunMulti (portal=0x292b490, isTopLevel=true, setHoldSnapshot=false, dest=0x2990dc8, altdest=0x2990dc8, completionTag=0x7ffdbc052d10 "") at pquery.c:1286
    #12 0x0000000000852b32 in PortalRun (portal=0x292b490, count=9223372036854775807, isTopLevel=true, run_once=true, dest=0x2990dc8, altdest=0x2990dc8, completionTag=0x7ffdbc052d10 "") at pquery.c:799
    #13 0x000000000084cebc in exec_simple_query (query_string=0x28c5ef0 "insert into t_insert values(9,'11','12','13');") at postgres.c:1122
    #14 0x0000000000850f3c in PostgresMain (argc=1, argv=0x28efaa8, dbname=0x28ef990 "testdb", username=0x28ef978 "xdb") at postgres.c:4153
    #15 0x00000000007c0168 in BackendRun (port=0x28e7970) at postmaster.c:4361
    #16 0x00000000007bf8fc in BackendStartup (port=0x28e7970) at postmaster.c:4033
    #17 0x00000000007bc139 in ServerLoop () at postmaster.c:1706
    #18 0x00000000007bb9f9 in PostmasterMain (argc=1, argv=0x28c0b60) at postmaster.c:1379
    #19 0x00000000006f19e8 in main (argc=1, argv=0x28c0b60) at main.c:228
    
    #使用next命令，单步调试
    (gdb) next
    207     if (phdr->pd_lower < SizeOfPageHeaderData ||
    (gdb) next
    208         phdr->pd_lower > phdr->pd_upper ||
    (gdb) next
    207     if (phdr->pd_lower < SizeOfPageHeaderData ||
    (gdb) next
    209         phdr->pd_upper > phdr->pd_special ||
    (gdb) next
    208         phdr->pd_lower > phdr->pd_upper ||
    (gdb) next
    210         phdr->pd_special > BLCKSZ)
    (gdb) next
    209         phdr->pd_upper > phdr->pd_special ||
    (gdb) next
    219     limit = OffsetNumberNext(PageGetMaxOffsetNumber(page));
    (gdb) next
    222     if (OffsetNumberIsValid(offsetNumber))
    (gdb) p offsetNumber
    $1 = 0
    #offsetNumber为0，没有指定插入的行偏移
    (gdb) next
    247         if (PageHasFreeLinePointers(phdr))
    #查看页头信息
    (gdb) p *phdr
    $2 = {pd_lsn = {xlogid = 1, xrecoff = 3677462648}, pd_checksum = 0, pd_flags = 0, pd_lower = 56, pd_upper = 7680, pd_special = 8192, pd_pagesize_version = 8196, pd_prune_xid = 1612849, 
      pd_linp = 0x7feaaefac318}
    (gdb) 
    #查看行指针信息
    (gdb) p phdr->pd_linp[0]
    $3 = {lp_off = 8128, lp_flags = 1, lp_len = 61}
    (gdb) p phdr->pd_linp[1]
    $4 = {lp_off = 8064, lp_flags = 1, lp_len = 61}
    ...
    (gdb) p phdr->pd_linp[8]
    $11 = {lp_off = 0, lp_flags = 0, lp_len = 0}
    #行指针偏移的lp_flags均为1（LP_NORMAL），表示正在使用。
    ...
    298     alignedSize = MAXALIGN(size);
    (gdb) p size
    $17 = 61
    (gdb) p alignedSize
    $18 = 5045956
    (gdb) next
    300     upper = (int) phdr->pd_upper - (int) alignedSize;
    (gdb) p alignedSize
    $19 = 64 #原为61，对齐为64
    (gdb) next
    302     if (lower > upper)
    (gdb) p lower
    $20 = 60
    (gdb) p upper
    $21 = 7616
    (gdb) next
    308     itemId = PageGetItemId(phdr, offsetNumber);
    (gdb) 
    310     if (needshuffle)
    (gdb) next
    315     ItemIdSetNormal(itemId, upper, size);
    (gdb) next
    332     memcpy((char *) page + upper, item, size);
    (gdb) p *itemId
    $23 = {lp_off = 7616, lp_flags = 1, lp_len = 61}
    ...
    (gdb) next
    338     return offsetNumber;
    (gdb) 
    #数据插入在行偏移为9（行指针下标=8）的位置上。
    (gdb) p offsetNumber
    $24 = 9
    (gdb) p phdr->pd_linp[8]
    $25 = {lp_off = 7616, lp_flags = 1, lp_len = 61}
    ...
    #说明：之所以新插入的数据没有放在2号偏移上面，是因为被删除的第2行没有被回收。
    #查看Page中的数据
    testdb=# select * from heap_page_items(get_raw_page('t_insert',0));
     lp | lp_off | lp_flags | lp_len | t_xmin  | t_xmax  | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |                                    t_data                                 
       
    ----+--------+----------+--------+---------+---------+----------+--------+-------------+------------+--------+--------+-------+---------------------------------------------------------------------------    ---      1 |   8128 |        1 |     61 | 1612841 |       0 |        0 | (0,1)  |           4 |       2306 |     24 |        |       | \x010000001731312020202020202020173132202020202020202017313320202020202020
    20
      2 |   8064 |        1 |     61 | 1612842 | 1612849 |        0 | (0,2)  |        8196 |        258 |     24 |        |       | \x020000001731312020202020202020173132202020202020202017313320202020202020
    20
      3 |   8000 |        1 |     61 | 1612843 |       0 |        0 | (0,3)  |           4 |       2306 |     24 |        |       | \x030000001731312020202020202020173132202020202020202017313320202020202020
    20
      4 |   7936 |        1 |     61 | 1612844 |       0 |        0 | (0,4)  |           4 |       2306 |     24 |        |       | \x040000001731312020202020202020173132202020202020202017313320202020202020
    20
      5 |   7872 |        1 |     61 | 1612845 |       0 |        0 | (0,5)  |           4 |       2306 |     24 |        |       | \x050000001731312020202020202020173132202020202020202017313320202020202020
    20
      6 |   7808 |        1 |     61 | 1612846 |       0 |        0 | (0,6)  |           4 |       2306 |     24 |        |       | \x060000001731312020202020202020173132202020202020202017313320202020202020
    20
      7 |   7744 |        1 |     61 | 1612847 |       0 |        0 | (0,7)  |           4 |       2306 |     24 |        |       | \x070000001731312020202020202020173132202020202020202017313320202020202020
    20
      8 |   7680 |        1 |     61 | 1612848 |       0 |        0 | (0,8)  |           4 |       2306 |     24 |        |       | \x080000001731312020202020202020173132202020202020202017313320202020202020
    20
      9 |   7616 |        1 |     61 | 1612850 |       0 |        0 | (0,9)  |           4 |       2050 |     24 |        |       | \x090000001731312020202020202020173132202020202020202017313320202020202020
    20
    (9 rows)
    
    

### 四、小结

得益于PG的开放性，可以在源代码层次深入了解其实现。  
知识要点：  
1、基本了解PageAddItemExtended函数的实现逻辑；  
2、基本懂得使用gdb辅助跟踪分析实现逻辑。  
下一节，按照自下往上的分析思路，将会解析RelationPutHeapTuple 函数

