这一小节继续介绍查询物理优化中的create_index_paths->create_bitmap_heap_path函数,该函数创建位图堆扫描访问路径节点。  
关于BitmapHeapScan的相关知识,请参照[PostgreSQL DBA(6) - SeqScan vs IndexScan vs
BitmapHeapScan](https://www.jianshu.com/p/4f819a43882d)这篇文章.  
本节没有描述具体的Cost成本计算方法(公式)，后续再行详述。

### 一、数据结构

**Cost相关**  
注意:实际使用的参数值通过系统配置文件定义,而不是这里的常量定义!

    
    
     typedef double Cost; /* execution cost (in page-access units) */
    
     /* defaults for costsize.c's Cost parameters */
     /* NB: cost-estimation code should use the variables, not these constants! */
     /* 注意:实际值通过系统配置文件定义,而不是这里的常量定义! */
     /* If you change these, update backend/utils/misc/postgresql.sample.conf */
     #define DEFAULT_SEQ_PAGE_COST  1.0       //顺序扫描page的成本
     #define DEFAULT_RANDOM_PAGE_COST  4.0      //随机扫描page的成本
     #define DEFAULT_CPU_TUPLE_COST  0.01     //处理一个元组的CPU成本
     #define DEFAULT_CPU_INDEX_TUPLE_COST 0.005   //处理一个索引元组的CPU成本
     #define DEFAULT_CPU_OPERATOR_COST  0.0025    //执行一次操作或函数的CPU成本
     #define DEFAULT_PARALLEL_TUPLE_COST 0.1    //并行执行,从一个worker传输一个元组到另一个worker的成本
     #define DEFAULT_PARALLEL_SETUP_COST  1000.0  //构建并行执行环境的成本
     
     #define DEFAULT_EFFECTIVE_CACHE_SIZE  524288    /*先前已有介绍, measured in pages */
    
     double      seq_page_cost = DEFAULT_SEQ_PAGE_COST;
     double      random_page_cost = DEFAULT_RANDOM_PAGE_COST;
     double      cpu_tuple_cost = DEFAULT_CPU_TUPLE_COST;
     double      cpu_index_tuple_cost = DEFAULT_CPU_INDEX_TUPLE_COST;
     double      cpu_operator_cost = DEFAULT_CPU_OPERATOR_COST;
     double      parallel_tuple_cost = DEFAULT_PARALLEL_TUPLE_COST;
     double      parallel_setup_cost = DEFAULT_PARALLEL_SETUP_COST;
     
     int         effective_cache_size = DEFAULT_EFFECTIVE_CACHE_SIZE;
     Cost        disable_cost = 1.0e10;//1后面10个0,通过设置一个巨大的成本,让优化器自动放弃此路径
     
     int         max_parallel_workers_per_gather = 2;//每次gather使用的worker数
    

### 二、源码解读

**create_bitmap_heap_path函数**  
create_index_paths- >create_bitmap_heap_path函数,创建位图堆扫描访问路径节点.

    
    
     /*
      * create_bitmap_heap_path
      *    Creates a path node for a bitmap scan.
      *    创建位图堆扫描访问路径节点
      *
      * 'bitmapqual' is a tree of IndexPath, BitmapAndPath, and BitmapOrPath nodes.
      * bitmapqual-IndexPath, BitmapAndPath, and BitmapOrPath节点组成的树
      * 'required_outer' is the set of outer relids for a parameterized path.
      * required_outer-参数化路径中依赖的外部relids
      * 'loop_count' is the number of repetitions of the indexscan to factor into
      *      estimates of caching behavior.
      * loop_count-上一节已介绍
      *
      * loop_count should match the value used when creating the component
      * IndexPaths.
      */
     BitmapHeapPath *
     create_bitmap_heap_path(PlannerInfo *root,
                             RelOptInfo *rel,
                             Path *bitmapqual,
                             Relids required_outer,
                             double loop_count,
                             int parallel_degree)
     {
         BitmapHeapPath *pathnode = makeNode(BitmapHeapPath);//创建节点
     
         pathnode->path.pathtype = T_BitmapHeapScan;
         pathnode->path.parent = rel;
         pathnode->path.pathtarget = rel->reltarget;
         pathnode->path.param_info = get_baserel_parampathinfo(root, rel,
                                                               required_outer);
         pathnode->path.parallel_aware = parallel_degree > 0 ? true : false;
         pathnode->path.parallel_safe = rel->consider_parallel;
         pathnode->path.parallel_workers = parallel_degree;
         pathnode->path.pathkeys = NIL;  /* always unordered */
     
         pathnode->bitmapqual = bitmapqual;
     
         cost_bitmap_heap_scan(&pathnode->path, root, rel,
                               pathnode->path.param_info,
                               bitmapqual, loop_count);//成本估算
     
         return pathnode;//返回结果
     }
     
    //-------------------------------------------------------- cost_bitmap_heap_scan
     /*
      * cost_bitmap_heap_scan
      *    Determines and returns the cost of scanning a relation using a bitmap
      *    index-then-heap plan.
      *    确定并返回使用BitmapIndexScan和BitmapHeapScan的成本.
      *
      * 'baserel' is the relation to be scanned
      * baserel-需扫描的Relation
      * 'param_info' is the ParamPathInfo if this is a parameterized path, else NULL
      * param_info-参数化信息
      * 'bitmapqual' is a tree of IndexPaths, BitmapAndPaths, and BitmapOrPaths
      * bitmapqual-位图条件表达式,IndexPath, BitmapAndPath, and BitmapOrPath节点组成的树
      * 'loop_count' is the number of repetitions of the indexscan to factor into
      *      estimates of caching behavior
      *
      * Note: the component IndexPaths in bitmapqual should have been costed
      * using the same loop_count.
      */
     void
     cost_bitmap_heap_scan(Path *path, PlannerInfo *root, RelOptInfo *baserel,
                           ParamPathInfo *param_info,
                           Path *bitmapqual, double loop_count)
     {
         Cost        startup_cost = 0;//启动成本
         Cost        run_cost = 0;//执行成本
         Cost        indexTotalCost;//索引扫描总成本
         QualCost    qpqual_cost;//表达式成本
         Cost        cpu_per_tuple;
         Cost        cost_per_page;
         Cost        cpu_run_cost;
         double      tuples_fetched;
         double      pages_fetched;
         double      spc_seq_page_cost,
                     spc_random_page_cost;
         double      T;
     
         /* Should only be applied to base relations */
         Assert(IsA(baserel, RelOptInfo));
         Assert(baserel->relid > 0);
         Assert(baserel->rtekind == RTE_RELATION);
     
         /* Mark the path with the correct row estimate */
         if (param_info)
             path->rows = param_info->ppi_rows;
         else
             path->rows = baserel->rows;
     
         if (!enable_bitmapscan)//不允许位图扫描
             startup_cost += disable_cost;//禁用之
     
         pages_fetched = compute_bitmap_pages(root, baserel, bitmapqual,
                                              loop_count, &indexTotalCost,
                                              &tuples_fetched);//计算页面数
     
         startup_cost += indexTotalCost;//启动成本为BitmapIndexScan的总成本
         T = (baserel->pages > 1) ? (double) baserel->pages : 1.0;//页面数
     
         /* Fetch estimated page costs for tablespace containing table. */
         get_tablespace_page_costs(baserel->reltablespace,
                                   &spc_random_page_cost,
                                   &spc_seq_page_cost);//访问表空间页面成本
     
         /*
          * For small numbers of pages we should charge spc_random_page_cost
          * apiece, while if nearly all the table's pages are being read, it's more
          * appropriate to charge spc_seq_page_cost apiece.  The effect is
          * nonlinear, too. For lack of a better idea, interpolate like this to
          * determine the cost per page.
          * 对于少量的页面，每个页面的成本为spc_random_page_cost，
          * 而如果几乎所有的页面都被读取，则每个页面的成本为spc_seq_page_cost。
          * 这种影响也是非线性的。由于缺乏更好的方法，通过插值法确定每页的成本。
          */
         if (pages_fetched >= 2.0)
             cost_per_page = spc_random_page_cost -                 (spc_random_page_cost - spc_seq_page_cost)
                 * sqrt(pages_fetched / T);
         else
             cost_per_page = spc_random_page_cost;
     
         run_cost += pages_fetched * cost_per_page;//执行成本
     
         /*
          * Estimate CPU costs per tuple.
          * 为每个元组估算CPU成本(Rechck步骤的成本)
          * 
          * Often the indexquals don't need to be rechecked at each tuple ... but
          * not always, especially not if there are enough tuples involved that the
          * bitmaps become lossy.  For the moment, just assume they will be
          * rechecked always.  This means we charge the full freight for all the
          * scan clauses. 
          * 通常情况下，索引约束条件不需要在每个元组上重新检查,但现实并非如此理想，
          * 尤其是当涉及到较多的元组时。就目前而言，
          * 优化器会假设它们总是会被重新检查。这意味着我们需要为所有扫描条件计算成本。
          */
         get_restriction_qual_cost(root, baserel, param_info, &qpqual_cost);//获取条件表达式
     
         startup_cost += qpqual_cost.startup;//增加启动成本
         cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple;//增加处理每个元组的CPU成本
         cpu_run_cost = cpu_per_tuple * tuples_fetched;//CPU运行成本
     
         /* Adjust costing for parallelism, if used. */
         if (path->parallel_workers > 0)//是否并行?
         {
             double      parallel_divisor = get_parallel_divisor(path);
     
             /* The CPU cost is divided among all the workers. */
             cpu_run_cost /= parallel_divisor;
     
             path->rows = clamp_row_est(path->rows / parallel_divisor);
         }
     
         //计算最终成本
         run_cost += cpu_run_cost;
     
         /* tlist eval costs are paid per output row, not per tuple scanned */
         startup_cost += path->pathtarget->cost.startup;
         run_cost += path->pathtarget->cost.per_tuple * path->rows;
     
         path->startup_cost = startup_cost;
         path->total_cost = startup_cost + run_cost;
     }
    
    //--------------------------------------- compute_bitmap_pages
    
     /*
      * compute_bitmap_pages
      * 
      * compute number of pages fetched from heap in bitmap heap scan.
      * 计算页面数
      */
     double
     compute_bitmap_pages(PlannerInfo *root, RelOptInfo *baserel, Path *bitmapqual,
                          int loop_count, Cost *cost, double *tuple)
     {
         Cost        indexTotalCost;
         Selectivity indexSelectivity;
         double      T;
         double      pages_fetched;
         double      tuples_fetched;
         double      heap_pages;
         long        maxentries;
     
         /*
          * Fetch total cost of obtaining the bitmap, as well as its total
          * selectivity.
          * 获取位图的总成本，以及它的总选择性。
          */
         cost_bitmap_tree_node(bitmapqual, &indexTotalCost, &indexSelectivity);
     
         /*
          * Estimate number of main-table pages fetched.
          * 估算主表的页面数
          */
         tuples_fetched = clamp_row_est(indexSelectivity * baserel->tuples);//计算总元组数
     
         T = (baserel->pages > 1) ? (double) baserel->pages : 1.0;
     
         /*
          * For a single scan, the number of heap pages that need to be fetched is
          * the same as the Mackert and Lohman formula for the case T <= b (ie, no
          * re-reads needed).
          * 对于单个扫描，需要获取的堆页面数量与T <= b(即不需要重新读取)的Mackert和Lohman公式相同。
          */
         pages_fetched = (2.0 * T * tuples_fetched) / (2.0 * T + tuples_fetched);
     
         /*
          * Calculate the number of pages fetched from the heap.  Then based on
          * current work_mem estimate get the estimated maxentries in the bitmap.
          * (Note that we always do this calculation based on the number of pages
          * that would be fetched in a single iteration, even if loop_count > 1.
          * That's correct, because only that number of entries will be stored in
          * the bitmap at one time.)
          * 计算从堆中读取的页面数.
          * 根据当前的work_mem估算得到位图中粗略的最大访问入口(entries)。
          * (请注意，我们总是根据单个迭代中获取的页面数来进行计算，
          * 即使loop_count > 1也是如此。因为只有该数量的条目在位图中只存储一次。
          */
         heap_pages = Min(pages_fetched, baserel->pages);//堆页面数
         maxentries = tbm_calculate_entries(work_mem * 1024L);//位图最大入口数
     
         if (loop_count > 1)
         {
             /*
              * For repeated bitmap scans, scale up the number of tuples fetched in
              * the Mackert and Lohman formula by the number of scans, so that we
              * estimate the number of pages fetched by all the scans. Then
              * pro-rate for one scan.
              */
             pages_fetched = index_pages_fetched(tuples_fetched * loop_count,
                                                 baserel->pages,
                                                 get_indexpath_pages(bitmapqual),
                                                 root);
             pages_fetched /= loop_count;
         }
     
         if (pages_fetched >= T)
             pages_fetched = T;//数据字典中的页面数
         else
             pages_fetched = ceil(pages_fetched);
     
         if (maxentries < heap_pages)//最大入口数小于堆页面数
         {
             double      exact_pages;
             double      lossy_pages;
     
             /*
              * Crude approximation of the number of lossy pages.  Because of the
              * way tbm_lossify() is coded, the number of lossy pages increases
              * very sharply as soon as we run short of memory; this formula has
              * that property and seems to perform adequately in testing, but it's
              * possible we could do better somehow.
              * 粗略估计缺页的数目。由于tbm_lossify()的编码方式，
              * 一旦内存不足，缺页的数量就会急剧增加;
              * 这个公式有这个性质，在测试中表现得很好，但有可能做得更好。
              */
             lossy_pages = Max(0, heap_pages - maxentries / 2);
             exact_pages = heap_pages - lossy_pages;
     
             /*
              * If there are lossy pages then recompute the  number of tuples
              * processed by the bitmap heap node.  We assume here that the chance
              * of a given tuple coming from an exact page is the same as the
              * chance that a given page is exact.  This might not be true, but
              * it's not clear how we can do any better.
              * 如果存在缺页面，则重新计算位图堆节点处理的元组数量。
              * 这里假设给定元组来自精确页面的概率与给定页面的概率相同。
              * 但这可能不符合实际情况，但我们不清楚如何才能做得更好:(
              */
             if (lossy_pages > 0)
                 tuples_fetched =
                     clamp_row_est(indexSelectivity *
                                   (exact_pages / heap_pages) * baserel->tuples +
                                   (lossy_pages / heap_pages) * baserel->tuples);
         }
     
         if (cost)
             *cost = indexTotalCost;
         if (tuple)
             *tuple = tuples_fetched;
     
         return pages_fetched;
     }
    
    //--------------------------- tbm_calculate_entries
    
     /*
      * tbm_calculate_entries
      *
      * Estimate number of hashtable entries we can have within maxbytes.
      */
     long
     tbm_calculate_entries(double maxbytes)
     {
         long        nbuckets;
     
         /*
          * Estimate number of hashtable entries we can have within maxbytes. This
          * estimates the hash cost as sizeof(PagetableEntry), which is good enough
          * for our purpose.  Also count an extra Pointer per entry for the arrays
          * created during iteration readout.
          * 估计maxbytes中可以包含的哈希表条目的数量。
          * 这将散列成本估计为sizeof(PagetableEntry)，这已经足够好了。
          * 还要为迭代读出期间创建的数组中每个条目计算额外的指针。
          */
         nbuckets = maxbytes /
             (sizeof(PagetableEntry) + sizeof(Pointer) + sizeof(Pointer));//桶数
         nbuckets = Min(nbuckets, INT_MAX - 1);  /* safety limit */
         nbuckets = Max(nbuckets, 16);   /* sanity limit */
     
         return nbuckets;
     }
    
    //--------------------------- cost_bitmap_tree_node
    
     /*
      * cost_bitmap_tree_node
      *      Extract cost and selectivity from a bitmap tree node (index/and/or)
      */
     void
     cost_bitmap_tree_node(Path *path, Cost *cost, Selectivity *selec)
     {
         if (IsA(path, IndexPath))//索引访问路径
         {
             *cost = ((IndexPath *) path)->indextotalcost;
             *selec = ((IndexPath *) path)->indexselectivity;
     
             /*
              * Charge a small amount per retrieved tuple to reflect the costs of
              * manipulating the bitmap.  This is mostly to make sure that a bitmap
              * scan doesn't look to be the same cost as an indexscan to retrieve a
              * single tuple.
              * 对每个检索到的元组计算少量成本，以反映操作位图的成本。
              * 这主要是为了确保位图扫描与索引扫描检索单个元组的成本不一样。
              */
             *cost += 0.1 * cpu_operator_cost * path->rows;
         }
         else if (IsA(path, BitmapAndPath))//BitmapAndPath
         {
             *cost = path->total_cost;
             *selec = ((BitmapAndPath *) path)->bitmapselectivity;
         }
         else if (IsA(path, BitmapOrPath))//BitmapOrPath
         {
             *cost = path->total_cost;
             *selec = ((BitmapOrPath *) path)->bitmapselectivity;
         }
         else
         {
             elog(ERROR, "unrecognized node type: %d", nodeTag(path));
             *cost = *selec = 0;     /* keep compiler quiet */
         }
     }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    select t1.* 
    from t_dwxx t1 
    where dwbh > '10000' and dwbh < '30000';
    

启动gdb跟踪

    
    
    (gdb) b create_bitmap_heap_path
    Breakpoint 1 at 0x78f1c1: file pathnode.c, line 1090.
    (gdb) c
    Continuing.
    
    Breakpoint 1, create_bitmap_heap_path (root=0x23d93d8, rel=0x248a788, bitmapqual=0x2473a08, required_outer=0x0, 
        loop_count=1, parallel_degree=0) at pathnode.c:1090
    1090        BitmapHeapPath *pathnode = makeNode(BitmapHeapPath);
    

创建节点,并赋值

    
    
    1090        BitmapHeapPath *pathnode = makeNode(BitmapHeapPath);
    (gdb) n
    1092        pathnode->path.pathtype = T_BitmapHeapScan;
    (gdb) n
    1093        pathnode->path.parent = rel;
    (gdb) n
    1094        pathnode->path.pathtarget = rel->reltarget;
    (gdb) n
    1095        pathnode->path.param_info = get_baserel_parampathinfo(root, rel,
    (gdb) 
    1097        pathnode->path.parallel_aware = parallel_degree > 0 ? true : false;
    (gdb) 
    1098        pathnode->path.parallel_safe = rel->consider_parallel;
    (gdb) 
    1099        pathnode->path.parallel_workers = parallel_degree;
    (gdb) 
    1100        pathnode->path.pathkeys = NIL;  /* always unordered */
    (gdb) 
    1102        pathnode->bitmapqual = bitmapqual;
    

进入cost_bitmap_heap_scan函数

    
    
    (gdb) 
    1104        cost_bitmap_heap_scan(&pathnode->path, root, rel,
    (gdb) step
    cost_bitmap_heap_scan (path=0x24737d8, root=0x23d93d8, baserel=0x248a788, param_info=0x0, bitmapqual=0x2473a08, 
        loop_count=1) at costsize.c:949
    949     Cost        startup_cost = 0;
    

输入参数,其中bitmapqual为T_IndexPath节点  
路径的其他关键信息:rows = 2223, startup_cost = 0.28500000000000003, total_cost =
169.23871600907944

    
    
    (gdb) p *(IndexPath *)bitmapqual
    $2 = {path = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x248a788, pathtarget = 0x248a998, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 2223, startup_cost = 0.28500000000000003, 
        total_cost = 169.23871600907944, pathkeys = 0x0}, indexinfo = 0x23b63b8, indexclauses = 0x2473948, 
      indexquals = 0x2473b38, indexqualcols = 0x2473b88, indexorderbys = 0x0, indexorderbycols = 0x0, 
      indexscandir = ForwardScanDirection, indextotalcost = 50.515000000000001, indexselectivity = 0.22227191011235958}
    

开始计算成本

    
    
    ...
    980     startup_cost += indexTotalCost;
    (gdb) p indexTotalCost
    $16 = 51.070750000000004
    (gdb) p startup_cost
    $17 = 0
    (gdb) p pages_fetched
    $18 = 64
    (gdb) p baserel->pages
    $19 = 64
    ...
    (gdb) p qpqual_cost
    $20 = {startup = 0, per_tuple = 0.0050000000000000001}
    

最终的访问路径信息

    
    
    (gdb) p *(BitmapHeapPath *)path
    $22 = {path = {type = T_BitmapHeapPath, pathtype = T_BitmapHeapScan, parent = 0x248a788, pathtarget = 0x248a998, 
        param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 2223, 
        startup_cost = 51.070750000000004, total_cost = 148.41575, pathkeys = 0x0}, bitmapqual = 0x2473a08}
    

除了BitmapHeapPath,还有BitmapOr和BitmapAnd,这两种Path的解析后续再详述.

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

