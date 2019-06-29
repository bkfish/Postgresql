这一小节主要介绍函数build_index_paths中的子函数create_index_path，该函数实现了索引扫描成本的估算主逻辑。

### 一、数据结构

**IndexOptInfo**  
回顾IndexOptInfo索引信息结构体

    
    
     typedef struct IndexOptInfo
     {
         NodeTag     type;
     
         Oid         indexoid;       /* Index的OID,OID of the index relation */
         Oid         reltablespace;  /* Index的表空间,tablespace of index (not table) */
         RelOptInfo *rel;            /* 指向Relation的指针,back-link to index's table */
     
         /* index-size statistics (from pg_class and elsewhere) */
         BlockNumber pages;          /* Index的pages,number of disk pages in index */
         double      tuples;         /* Index的元组数,number of index tuples in index */
         int         tree_height;    /* 索引高度,index tree height, or -1 if unknown */
     
         /* index descriptor information */
         int         ncolumns;       /* 索引的列数,number of columns in index */
         int         nkeycolumns;    /* 索引的关键列数,number of key columns in index */
         int        *indexkeys;      /* column numbers of index's attributes both
                                      * key and included columns, or 0 */
         Oid        *indexcollations;    /* OIDs of collations of index columns */
         Oid        *opfamily;       /* OIDs of operator families for columns */
         Oid        *opcintype;      /* OIDs of opclass declared input data types */
         Oid        *sortopfamily;   /* OIDs of btree opfamilies, if orderable */
         bool       *reverse_sort;   /* 倒序?is sort order descending? */
         bool       *nulls_first;    /* NULLs值优先?do NULLs come first in the sort order? */
         bool       *canreturn;      /* 索引列可通过Index-Only Scan返回?which index cols can be returned in an
                                      * index-only scan? */
         Oid         relam;          /* 访问方法OID,OID of the access method (in pg_am) */
     
         List       *indexprs;       /* 非简单索引列表达式链表,如函数索引,expressions for non-simple index columns */
         List       *indpred;        /* 部分索引的谓词链表,predicate if a partial index, else NIL */
     
         List       *indextlist;     /* 索引列(TargetEntry结构体链表),targetlist representing index columns */
     
         List       *indrestrictinfo;    /* 父关系的baserestrictinfo列表，
                                          * 不包含索引谓词隐含的所有条件
                                          * (除非是目标rel，请参阅check_index_predicates()中的注释),
                                          * parent relation's baserestrictinfo
                                          * list, less any conditions implied by
                                          * the index's predicate (unless it's a
                                          * target rel, see comments in
                                          * check_index_predicates()) */
     
         bool        predOK;         /* True,如索引谓词满足查询要求,true if index predicate matches query */
         bool        unique;         /* 是否唯一索引,true if a unique index */
         bool        immediate;      /* 唯一性校验是否立即生效,is uniqueness enforced immediately? */
         bool        hypothetical;   /* 是否虚拟索引,true if index doesn't really exist */
     
         /* Remaining fields are copied from the index AM's API struct: */
         //从Index Relation拷贝过来的AM(访问方法)API信息
         bool        amcanorderbyop; /* does AM support order by operator result? */
         bool        amoptionalkey;  /* can query omit key for the first column? */
         bool        amsearcharray;  /* can AM handle ScalarArrayOpExpr quals? */
         bool        amsearchnulls;  /* can AM search for NULL/NOT NULL entries? */
         bool        amhasgettuple;  /* does AM have amgettuple interface? */
         bool        amhasgetbitmap; /* does AM have amgetbitmap interface? */
         bool        amcanparallel;  /* does AM support parallel scan? */
         /* Rather than include amapi.h here, we declare amcostestimate like this */
         void        (*amcostestimate) ();   /* 访问方法的估算函数,AM's cost estimator */
     } IndexOptInfo;
     
    

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

**create_index_path**  
该函数创建索引扫描路径节点,其中调用函数cost_index计算索引扫描成本.

    
    
    //----------------------------------------------- create_index_path
    
     /*
      * create_index_path
      *    Creates a path node for an index scan.
      *    创建索引扫描路径节点
      *
      * 'index' is a usable index.
      * 'indexclauses' is a list of RestrictInfo nodes representing clauses
      *          to be used as index qual conditions in the scan.
      * 'indexclausecols' is an integer list of index column numbers (zero based)
      *          the indexclauses can be used with.
      * 'indexorderbys' is a list of bare expressions (no RestrictInfos)
      *          to be used as index ordering operators in the scan.
      * 'indexorderbycols' is an integer list of index column numbers (zero based)
      *          the ordering operators can be used with.
      * 'pathkeys' describes the ordering of the path.
      * 'indexscandir' is ForwardScanDirection or BackwardScanDirection
      *          for an ordered index, or NoMovementScanDirection for
      *          an unordered index.
      * 'indexonly' is true if an index-only scan is wanted.
      * 'required_outer' is the set of outer relids for a parameterized path.
      * 'loop_count' is the number of repetitions of the indexscan to factor into
      *      estimates of caching behavior.
      * 'partial_path' is true if constructing a parallel index scan path.
      *
      * Returns the new path node.
      */
     IndexPath *
     create_index_path(PlannerInfo *root,//优化器信息
                       IndexOptInfo *index,//索引信息
                       List *indexclauses,//索引约束条件链表
                       List *indexclausecols,//索引约束条件列编号链表,与indexclauses一一对应
                       List *indexorderbys,//ORDER BY原始表达式链表
                       List *indexorderbycols,//ORDER BY列编号链表
                       List *pathkeys,//排序路径键
                       ScanDirection indexscandir,//扫描方向
                       bool indexonly,//纯索引扫描?
                       Relids required_outer,//需依赖的外部Relids
                       double loop_count,//用于估计缓存的重复次数
                       bool partial_path)//是否并行索引扫描
     {
         IndexPath  *pathnode = makeNode(IndexPath);//构建节点
         RelOptInfo *rel = index->rel;//索引对应的Rel
         List       *indexquals,
                    *indexqualcols;
     
         pathnode->path.pathtype = indexonly ? T_IndexOnlyScan : T_IndexScan;//路径类型
         pathnode->path.parent = rel;//Relation
         pathnode->path.pathtarget = rel->reltarget;//路径最终的投影列
         pathnode->path.param_info = get_baserel_parampathinfo(root, rel,
                                                               required_outer);//参数化信息
         pathnode->path.parallel_aware = false;//
         pathnode->path.parallel_safe = rel->consider_parallel;//是否并行
         pathnode->path.parallel_workers = 0;//worker数目
         pathnode->path.pathkeys = pathkeys;//排序路径键
     
         /* Convert clauses to  the executor can handle */
         //转换条件子句(clauses)为执行器可处理的索引表达式(indexquals)
         expand_indexqual_conditions(index, indexclauses, indexclausecols,
                                     &indexquals, &indexqualcols);
     
         /* 填充路径节点信息,Fill in the pathnode */
         pathnode->indexinfo = index;
         pathnode->indexclauses = indexclauses;
         pathnode->indexquals = indexquals;
         pathnode->indexqualcols = indexqualcols;
         pathnode->indexorderbys = indexorderbys;
         pathnode->indexorderbycols = indexorderbycols;
         pathnode->indexscandir = indexscandir;
     
         cost_index(pathnode, root, loop_count, partial_path);//估算成本
     
         return pathnode;
     }
    
    
    //------------------------------------ expand_indexqual_conditions
    
     /*
      * expand_indexqual_conditions
      *    Given a list of RestrictInfo nodes, produce a list of directly usable
      *    index qual clauses.
      *    给定RestrictInfo节点(约束条件),产生直接可用的索引表达式子句
      *
      * Standard qual clauses (those in the index's opfamily) are passed through
      * unchanged.  Boolean clauses and "special" index operators are expanded
      * into clauses that the indexscan machinery will know what to do with.
      * RowCompare clauses are simplified if necessary to create a clause that is
      * fully checkable by the index.
      * 标准的条件子句(位于索引opfamily中)可不作修改直接使用.
      * 布尔子句和"special"索引操作符扩展为索引扫描执行器可以处理的子句.
      * 如需要，RowCompare子句将简化为可由索引完全检查的子句。
      * 
      * In addition to the expressions themselves, there are auxiliary lists
      * of the index column numbers that the clauses are meant to be used with;
      * we generate an updated column number list for the result.  (This is not
      * the identical list because one input clause sometimes produces more than
      * one output clause.)
      * 除了表达式本身之外，还有索引列号的辅助链表，这些子句将使用这些列号.这些列号
      * 将被更新用于结果返回(不是相同的链表，因为一个输入子句有时会产生多个输出子句。).
      * 
      * The input clauses are sorted by column number, and so the output is too.
      * (This is depended on in various places in both planner and executor.)
      * 输入子句通过列号排序,输出子句也是如此.
      */
     void
     expand_indexqual_conditions(IndexOptInfo *index,
                                 List *indexclauses, List *indexclausecols,
                                 List **indexquals_p, List **indexqualcols_p)
     {
         List       *indexquals = NIL;
         List       *indexqualcols = NIL;
         ListCell   *lcc,
                    *lci;
     
         forboth(lcc, indexclauses, lci, indexclausecols)//扫描索引子句链表和匹配的列号
         {
             RestrictInfo *rinfo = (RestrictInfo *) lfirst(lcc);
             int         indexcol = lfirst_int(lci);
             Expr       *clause = rinfo->clause;//条件子句
             Oid         curFamily;
             Oid         curCollation;
     
             Assert(indexcol < index->nkeycolumns);
     
             curFamily = index->opfamily[indexcol];//索引列的opfamily
             curCollation = index->indexcollations[indexcol];//排序规则
     
             /* First check for boolean cases */
             if (IsBooleanOpfamily(curFamily))//布尔
             {
                 Expr       *boolqual;
     
                 boolqual = expand_boolean_index_clause((Node *) clause,
                                                        indexcol,
                                                        index);//布尔表达式
                 if (boolqual)
                 {
                     indexquals = lappend(indexquals,
                                          make_simple_restrictinfo(boolqual));//添加到结果中
                     indexqualcols = lappend_int(indexqualcols, indexcol);//列号
                     continue;
                 }
             }
     
             /*
              * Else it must be an opclause (usual case), ScalarArrayOp,
              * RowCompare, or NullTest
              */
             if (is_opclause(clause))//普通的操作符子句
             {
                 indexquals = list_concat(indexquals,
                                          expand_indexqual_opclause(rinfo,
                                                                    curFamily,
                                                                    curCollation));//合并到结果链表中
                 /* expand_indexqual_opclause can produce multiple clauses */
                 while (list_length(indexqualcols) < list_length(indexquals))
                     indexqualcols = lappend_int(indexqualcols, indexcol);
             }
             else if (IsA(clause, ScalarArrayOpExpr))//ScalarArrayOpExpr
             {
                 /* no extra work at this time */
                 indexquals = lappend(indexquals, rinfo);
                 indexqualcols = lappend_int(indexqualcols, indexcol);
             }
             else if (IsA(clause, RowCompareExpr))//RowCompareExpr
             {
                 indexquals = lappend(indexquals,
                                      expand_indexqual_rowcompare(rinfo,
                                                                  index,
                                                                  indexcol));
                 indexqualcols = lappend_int(indexqualcols, indexcol);
             }
             else if (IsA(clause, NullTest))//NullTest
             {
                 Assert(index->amsearchnulls);
                 indexquals = lappend(indexquals, rinfo);
                 indexqualcols = lappend_int(indexqualcols, indexcol);
             }
             else
                 elog(ERROR, "unsupported indexqual type: %d",
                      (int) nodeTag(clause));
         }
     
         *indexquals_p = indexquals;//结果赋值
         *indexqualcols_p = indexqualcols;
     }
     
    //------------------------------------ cost_index
     /*
      * cost_index
      *    Determines and returns the cost of scanning a relation using an index.
      *    确定和返回索引扫描的成本 
      *
      * 'path' describes the indexscan under consideration, and is complete
      *      except for the fields to be set by this routine
      * path-位于考虑之列的索引扫描路径,除了本例程要设置的字段外其他信息已完整.
      *
      * 'loop_count' is the number of repetitions of the indexscan to factor into
      *      estimates of caching behavior
      * loop_count-用于估计缓存的重复次数
      *
      * In addition to rows, startup_cost and total_cost, cost_index() sets the
      * path's indextotalcost and indexselectivity fields.  These values will be
      * needed if the IndexPath is used in a BitmapIndexScan.
      * 除了行、startup_cost和total_cost之外，函数cost_index
      * 还设置了访问路径的indextotalcost和indexselectivity字段。
      * 如果在位图索引扫描中使用IndexPath，则需要这些值。
      *
      * NOTE: path->indexquals must contain only clauses usable as index
      * restrictions.  Any additional quals evaluated as qpquals may reduce the
      * number of returned tuples, but they won't reduce the number of tuples
      * we have to fetch from the table, so they don't reduce the scan cost.
      * 注意:path->indexquals必须仅包含用作索引约束条件的子句。任何作为qpquals评估的
      * 额外条件可能会减少返回元组的数量，但它们不会减少必须从表中获取的元组
      * 数量，因此它们不会降低扫描成本。
      */
     void
     cost_index(IndexPath *path, PlannerInfo *root, double loop_count,
                bool partial_path)
     {
         IndexOptInfo *index = path->indexinfo;//索引信息
         RelOptInfo *baserel = index->rel;//RelOptInfo信息
         bool        indexonly = (path->path.pathtype == T_IndexOnlyScan);//是否纯索引扫描
         amcostestimate_function amcostestimate;//索引访问方法成本估算函数
         List       *qpquals;//qpquals链表
         Cost        startup_cost = 0;//启动成本
         Cost        run_cost = 0;//执行成本
         Cost        cpu_run_cost = 0;//cpu执行成本
         Cost        indexStartupCost;//索引启动成本
         Cost        indexTotalCost;//索引总成本
         Selectivity indexSelectivity;//选择率
         double      indexCorrelation,//
                     csquared;//
         double      spc_seq_page_cost,
                     spc_random_page_cost;
         Cost        min_IO_cost,//最小IO成本
                     max_IO_cost;//最大IO成本
         QualCost    qpqual_cost;//表达式成本
         Cost        cpu_per_tuple;//每个tuple处理成本
         double      tuples_fetched;//取得的元组数量
         double      pages_fetched;//取得的page数量
         double      rand_heap_pages;//随机访问的堆page数量
         double      index_pages;//索引page数量
     
         /* Should only be applied to base relations */
         Assert(IsA(baserel, RelOptInfo) &&
                IsA(index, IndexOptInfo));
         Assert(baserel->relid > 0);
         Assert(baserel->rtekind == RTE_RELATION);
     
         /*
          * Mark the path with the correct row estimate, and identify which quals
          * will need to be enforced as qpquals.  We need not check any quals that
          * are implied by the index's predicate, so we can use indrestrictinfo not
          * baserestrictinfo as the list of relevant restriction clauses for the
          * rel.
          */
         if (path->path.param_info)//存在参数化信息
         {
             path->path.rows = path->path.param_info->ppi_rows;
             /* qpquals come from the rel's restriction clauses and ppi_clauses */
             qpquals = list_concat(
                                   extract_nonindex_conditions(path->indexinfo->indrestrictinfo,
                                                               path->indexquals),
                                   extract_nonindex_conditions(path->path.param_info->ppi_clauses,
                                                               path->indexquals));
         }
         else
         {
             path->path.rows = baserel->rows;//基表的估算行数
             /* qpquals come from just the rel's restriction clauses */跑
             qpquals = extract_nonindex_conditions(path->indexinfo->indrestrictinfo,
                                                   path->indexquals);//从rel的约束条件子句中获取qpquals
         }
     
         if (!enable_indexscan)
             startup_cost += disable_cost;//禁用索引扫描
         /* we don't need to check enable_indexonlyscan; indxpath.c does that */
     
         /*
          * Call index-access-method-specific code to estimate the processing cost
          * for scanning the index, as well as the selectivity of the index (ie,
          * the fraction of main-table tuples we will have to retrieve) and its
          * correlation to the main-table tuple order.  We need a cast here because
          * relation.h uses a weak function type to avoid including amapi.h.
          */
         amcostestimate = (amcostestimate_function) index->amcostestimate;//索引访问路径成本估算函数
         amcostestimate(root, path, loop_count,
                        &indexStartupCost, &indexTotalCost,
                        &indexSelectivity, &indexCorrelation,
                        &index_pages);//调用函数btcostestimate
     
         /*
          * Save amcostestimate's results for possible use in bitmap scan planning.
          * We don't bother to save indexStartupCost or indexCorrelation, because a
          * bitmap scan doesn't care about either.
          */
         path->indextotalcost = indexTotalCost;//赋值
         path->indexselectivity = indexSelectivity;
     
         /* all costs for touching index itself included here */
         startup_cost += indexStartupCost;
         run_cost += indexTotalCost - indexStartupCost;
     
         /* estimate number of main-table tuples fetched */
         tuples_fetched = clamp_row_est(indexSelectivity * baserel->tuples);//取得的元组数量
     
         /* fetch estimated page costs for tablespace containing table */
         get_tablespace_page_costs(baserel->reltablespace,
                                   &spc_random_page_cost,
                                   &spc_seq_page_cost);//表空间访问page成本
     
         /*----------          * Estimate number of main-table pages fetched, and compute I/O cost.
          *
          * When the index ordering is uncorrelated with the table ordering,
          * we use an approximation proposed by Mackert and Lohman (see
          * index_pages_fetched() for details) to compute the number of pages
          * fetched, and then charge spc_random_page_cost per page fetched.
          *
          * When the index ordering is exactly correlated with the table ordering
          * (just after a CLUSTER, for example), the number of pages fetched should
          * be exactly selectivity * table_size.  What's more, all but the first
          * will be sequential fetches, not the random fetches that occur in the
          * uncorrelated case.  So if the number of pages is more than 1, we
          * ought to charge
          *      spc_random_page_cost + (pages_fetched - 1) * spc_seq_page_cost
          * For partially-correlated indexes, we ought to charge somewhere between
          * these two estimates.  We currently interpolate linearly between the
          * estimates based on the correlation squared (XXX is that appropriate?).
          *
          * If it's an index-only scan, then we will not need to fetch any heap
          * pages for which the visibility map shows all tuples are visible.
          * Hence, reduce the estimated number of heap fetches accordingly.
          * We use the measured fraction of the entire heap that is all-visible,
          * which might not be particularly relevant to the subset of the heap
          * that this query will fetch; but it's not clear how to do better.
          *----------          */
         if (loop_count > 1)//次数 > 1
         {
             /*
              * For repeated indexscans, the appropriate estimate for the
              * uncorrelated case is to scale up the number of tuples fetched in
              * the Mackert and Lohman formula by the number of scans, so that we
              * estimate the number of pages fetched by all the scans; then
              * pro-rate the costs for one scan.  In this case we assume all the
              * fetches are random accesses.
              */
             pages_fetched = index_pages_fetched(tuples_fetched * loop_count,
                                                 baserel->pages,
                                                 (double) index->pages,
                                                 root);
     
             if (indexonly)
                 pages_fetched = ceil(pages_fetched * (1.0 - baserel->allvisfrac));
     
             rand_heap_pages = pages_fetched;
     
             max_IO_cost = (pages_fetched * spc_random_page_cost) / loop_count;
     
             /*
              * In the perfectly correlated case, the number of pages touched by
              * each scan is selectivity * table_size, and we can use the Mackert
              * and Lohman formula at the page level to estimate how much work is
              * saved by caching across scans.  We still assume all the fetches are
              * random, though, which is an overestimate that's hard to correct for
              * without double-counting the cache effects.  (But in most cases
              * where such a plan is actually interesting, only one page would get
              * fetched per scan anyway, so it shouldn't matter much.)
              */
             pages_fetched = ceil(indexSelectivity * (double) baserel->pages);
     
             pages_fetched = index_pages_fetched(pages_fetched * loop_count,
                                                 baserel->pages,
                                                 (double) index->pages,
                                                 root);
     
             if (indexonly)
                 pages_fetched = ceil(pages_fetched * (1.0 - baserel->allvisfrac));
     
             min_IO_cost = (pages_fetched * spc_random_page_cost) / loop_count;
         }
         else //次数 <= 1
         {
             /*
              * Normal case: apply the Mackert and Lohman formula, and then
              * interpolate between that and the correlation-derived result.
              */
             pages_fetched = index_pages_fetched(tuples_fetched,
                                                 baserel->pages,
                                                 (double) index->pages,
                                                 root);//取得的page数量
     
             if (indexonly)
                 pages_fetched = ceil(pages_fetched * (1.0 - baserel->allvisfrac));//纯索引扫描
     
             rand_heap_pages = pages_fetched;//随机访问的堆page数量
     
             /* max_IO_cost is for the perfectly uncorrelated case (csquared=0) */
             //最大IO成本,假定所有的page都是随机访问获得(csquared=0)
             max_IO_cost = pages_fetched * spc_random_page_cost;
     
             /* min_IO_cost is for the perfectly correlated case (csquared=1) */
             //最小IO成本,假定索引和堆数据都是顺序存储(csquared=1)
             pages_fetched = ceil(indexSelectivity * (double) baserel->pages);
     
             if (indexonly)
                 pages_fetched = ceil(pages_fetched * (1.0 - baserel->allvisfrac));
     
             if (pages_fetched > 0)
             {
                 min_IO_cost = spc_random_page_cost;
                 if (pages_fetched > 1)
                     min_IO_cost += (pages_fetched - 1) * spc_seq_page_cost;
             }
             else
                 min_IO_cost = 0;
         }
     
         if (partial_path)//并行
         {
             /*
              * For index only scans compute workers based on number of index pages
              * fetched; the number of heap pages we fetch might be so small as to
              * effectively rule out parallelism, which we don't want to do.
              */
             if (indexonly)
                 rand_heap_pages = -1;
     
             /*
              * Estimate the number of parallel workers required to scan index. Use
              * the number of heap pages computed considering heap fetches won't be
              * sequential as for parallel scans the pages are accessed in random
              * order.
              */
             path->path.parallel_workers = compute_parallel_worker(baserel,
                                                                   rand_heap_pages,
                                                                   index_pages,
                                                                   max_parallel_workers_per_gather);
     
             /*
              * Fall out if workers can't be assigned for parallel scan, because in
              * such a case this path will be rejected.  So there is no benefit in
              * doing extra computation.
              */
             if (path->path.parallel_workers <= 0)
                 return;
     
             path->path.parallel_aware = true;
         }
     
         /*
          * Now interpolate based on estimated index order correlation to get total
          * disk I/O cost for main table accesses.
          * 根据估算的索引顺序关联来插值，以获得主表访问的总I/O成本
          */
         csquared = indexCorrelation * indexCorrelation;
     
         run_cost += max_IO_cost + csquared * (min_IO_cost - max_IO_cost);
     
         /*
          * Estimate CPU costs per tuple.
          * 估算处理每个元组的CPU成本
          *
          * What we want here is cpu_tuple_cost plus the evaluation costs of any
          * qual clauses that we have to evaluate as qpquals.
          */
         cost_qual_eval(&qpqual_cost, qpquals, root);
     
         startup_cost += qpqual_cost.startup;
         cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple;
     
         cpu_run_cost += cpu_per_tuple * tuples_fetched;
     
         /* tlist eval costs are paid per output row, not per tuple scanned */
         startup_cost += path->path.pathtarget->cost.startup;
         cpu_run_cost += path->path.pathtarget->cost.per_tuple * path->path.rows;
     
         /* Adjust costing for parallelism, if used. */
         if (path->path.parallel_workers > 0)
         {
             double      parallel_divisor = get_parallel_divisor(&path->path);
     
             path->path.rows = clamp_row_est(path->path.rows / parallel_divisor);
     
             /* The CPU cost is divided among all the workers. */
             cpu_run_cost /= parallel_divisor;
         }
     
         run_cost += cpu_run_cost;
     
         path->path.startup_cost = startup_cost;
         path->path.total_cost = startup_cost + run_cost;
     }
    
    
    //------------------------- btcostestimate
     void
     btcostestimate(PlannerInfo *root, IndexPath *path, double loop_count,
                    Cost *indexStartupCost, Cost *indexTotalCost,
                    Selectivity *indexSelectivity, double *indexCorrelation,
                    double *indexPages)
     {
         IndexOptInfo *index = path->indexinfo;
         List       *qinfos;
         GenericCosts costs;
         Oid         relid;
         AttrNumber  colnum;
         VariableStatData vardata;
         double      numIndexTuples;
         Cost        descentCost;
         List       *indexBoundQuals;
         int         indexcol;
         bool        eqQualHere;
         bool        found_saop;
         bool        found_is_null_op;
         double      num_sa_scans;
         ListCell   *lc;
     
         /* Do preliminary analysis of indexquals */
         qinfos = deconstruct_indexquals(path);//拆解路径,生成条件链表
     
         /*
          * For a btree scan, only leading '=' quals plus inequality quals for the
          * immediately next attribute contribute to index selectivity (these are
          * the "boundary quals" that determine the starting and stopping points of
          * the index scan).  Additional quals can suppress visits to the heap, so
          * it's OK to count them in indexSelectivity, but they should not count
          * for estimating numIndexTuples.  So we must examine the given indexquals
          * to find out which ones count as boundary quals.  We rely on the
          * knowledge that they are given in index column order.
          * 对于btree扫描，只有下一个属性的前导'='条件加上不等号条件
          * 有助于索引选择性(这些是确定索引扫描起始和停止的“边界条件”)。
          * 额外的条件可以抑制对堆数据的访问，所以在indexSelectivity中
          * 统计它们是可以的，但是它们不应该在估算索引元组数目(索引也是元组的一种)的时
          * 候统计。因此，必须检查给定的索引条件，以找出哪些被
          * 算作边界条件。需依赖索引信息给出的索引列顺序进行判断.
          *
          * For a RowCompareExpr, we consider only the first column, just as
          * rowcomparesel() does.
          *
          * If there's a ScalarArrayOpExpr in the quals, we'll actually perform N
          * index scans not one, but the ScalarArrayOpExpr's operator can be
          * considered to act the same as it normally does.
          */
         indexBoundQuals = NIL;//索引边界条件
         indexcol = 0;//索引列编号
         eqQualHere = false;//
         found_saop = false;
         found_is_null_op = false;
         num_sa_scans = 1;
         foreach(lc, qinfos)//遍历条件链表
         {
             IndexQualInfo *qinfo = (IndexQualInfo *) lfirst(lc);
             RestrictInfo *rinfo = qinfo->rinfo;
             Expr       *clause = rinfo->clause;
             Oid         clause_op;
             int         op_strategy;
     
             if (indexcol != qinfo->indexcol)//indexcol匹配才进行后续处理
             {
                 /* Beginning of a new column's quals */
                 if (!eqQualHere)
                     break;          /* done if no '=' qual for indexcol */
                 eqQualHere = false;
                 indexcol++;
                 if (indexcol != qinfo->indexcol)
                     break;          /* no quals at all for indexcol */
             }
     
             if (IsA(clause, ScalarArrayOpExpr))//ScalarArrayOpExpr
             {
                 int         alength = estimate_array_length(qinfo->other_operand);
     
                 found_saop = true;
                 /* count up number of SA scans induced by indexBoundQuals only */
                 if (alength > 1)
                     num_sa_scans *= alength;
             }
             else if (IsA(clause, NullTest))
             {
                 NullTest   *nt = (NullTest *) clause;
     
                 if (nt->nulltesttype == IS_NULL)
                 {
                     found_is_null_op = true;
                     /* IS NULL is like = for selectivity determination purposes */
                     eqQualHere = true;
                 }
             }
     
             /*
              * We would need to commute the clause_op if not varonleft, except
              * that we only care if it's equality or not, so that refinement is
              * unnecessary.
              */
             clause_op = qinfo->clause_op;
     
             /* check for equality operator */
             if (OidIsValid(clause_op))//普通的操作符
             {
                 op_strategy = get_op_opfamily_strategy(clause_op,
                                                        index->opfamily[indexcol]);
                 Assert(op_strategy != 0);   /* not a member of opfamily?? */
                 if (op_strategy == BTEqualStrategyNumber)
                     eqQualHere = true;
             }
     
             indexBoundQuals = lappend(indexBoundQuals, rinfo);
         }
     
         /*
          * If index is unique and we found an '=' clause for each column, we can
          * just assume numIndexTuples = 1 and skip the expensive
          * clauselist_selectivity calculations.  However, a ScalarArrayOp or
          * NullTest invalidates that theory, even though it sets eqQualHere.
          * 如果index是唯一的，并且我们为每个列找到了一个'='子句，那么可以
          * 假设numIndexTuples = 1，并跳过昂贵的clauselist_selectivity计算结果。
          * 这种判断不适用于ScalarArrayOp或NullTest。
          */
         if (index->unique &&
             indexcol == index->nkeycolumns - 1 &&
             eqQualHere &&
             !found_saop &&
             !found_is_null_op)
             numIndexTuples = 1.0;//唯一索引
         else//非唯一索引
         {
             List       *selectivityQuals;
             Selectivity btreeSelectivity;//选择率
     
             /*
              * If the index is partial, AND the index predicate with the
              * index-bound quals to produce a more accurate idea of the number of
              * rows covered by the bound conditions.
              */
             selectivityQuals = add_predicate_to_quals(index, indexBoundQuals);//添加谓词
     
             btreeSelectivity = clauselist_selectivity(root, selectivityQuals,
                                                       index->rel->relid,
                                                       JOIN_INNER,
                                                       NULL);//获取选择率
             numIndexTuples = btreeSelectivity * index->rel->tuples;//索引元组数目
     
             /*
              * As in genericcostestimate(), we have to adjust for any
              * ScalarArrayOpExpr quals included in indexBoundQuals, and then round
              * to integer.
              */
             numIndexTuples = rint(numIndexTuples / num_sa_scans);
         }
     
         /*
          * Now do generic index cost estimation.
          * 执行常规的索引成本估算
          */
         MemSet(&costs, 0, sizeof(costs));
         costs.numIndexTuples = numIndexTuples;
     
         genericcostestimate(root, path, loop_count, qinfos, &costs);
     
         /*
          * Add a CPU-cost component to represent the costs of initial btree
          * descent.  We don't charge any I/O cost for touching upper btree levels,
          * since they tend to stay in cache, but we still have to do about log2(N)
          * comparisons to descend a btree of N leaf tuples.  We charge one
          * cpu_operator_cost per comparison.
          * 添加一个cpu成本组件来表示初始化BTree树层次下降的成本。
          * BTree上层节点可以认为已存在于缓存中,因此不耗成本,但沿着树往下沉时,需要
          * 执行log2(N)次比较(N个叶子元组的BTree)。每次比较,成本为cpu_operator_cost
          * 
          * If there are ScalarArrayOpExprs, charge this once per SA scan.  The
          * ones after the first one are not startup cost so far as the overall
          * plan is concerned, so add them only to "total" cost.
          * 如存在ScalarArrayOpExprs,则每次SA扫描成本增加cpu_operator_cost
          */
         if (index->tuples > 1)      /* avoid computing log(0) */
         {
             descentCost = ceil(log(index->tuples) / log(2.0)) * cpu_operator_cost;
             costs.indexStartupCost += descentCost;
             costs.indexTotalCost += costs.num_sa_scans * descentCost;
         }
     
         /*
          * Even though we're not charging I/O cost for touching upper btree pages,
          * it's still reasonable to charge some CPU cost per page descended
          * through.  Moreover, if we had no such charge at all, bloated indexes
          * would appear to have the same search cost as unbloated ones, at least
          * in cases where only a single leaf page is expected to be visited.  This
          * cost is somewhat arbitrarily set at 50x cpu_operator_cost per page
          * touched.  The number of such pages is btree tree height plus one (ie,
          * we charge for the leaf page too).  As above, charge once per SA scan.
          * BTree树往下遍历时的成本descentCost=(树高+1)*50*cpu_operator_cost
          */
         descentCost = (index->tree_height + 1) * 50.0 * cpu_operator_cost;
         costs.indexStartupCost += descentCost;
         costs.indexTotalCost += costs.num_sa_scans * descentCost;
     
         /*
          * If we can get an estimate of the first column's ordering correlation C
          * from pg_statistic, estimate the index correlation as C for a
          * single-column index, or C * 0.75 for multiple columns. (The idea here
          * is that multiple columns dilute the importance of the first column's
          * ordering, but don't negate it entirely.  Before 8.0 we divided the
          * correlation by the number of columns, but that seems too strong.)
          * 如果我们可以从pg_statistical中得到第一列排序相关C的估计，那么对于单列索引，
          * 可以将索引相关性估计为C，对于多列，可以将其估计为C * 0.75。
          * (这里的想法是，多列淡化了第一列排序的重要性，但不要完全否定它。
          * 在8.0之前，我们将相关性除以列数，这种做法似乎太过了)。
          */
         MemSet(&vardata, 0, sizeof(vardata));
     
         if (index->indexkeys[0] != 0)
         {
             /* Simple variable --- look to stats for the underlying table */
             RangeTblEntry *rte = planner_rt_fetch(index->rel->relid, root);
     
             Assert(rte->rtekind == RTE_RELATION);
             relid = rte->relid;
             Assert(relid != InvalidOid);
             colnum = index->indexkeys[0];
     
             if (get_relation_stats_hook &&
                 (*get_relation_stats_hook) (root, rte, colnum, &vardata))
             {
                 /*
                  * The hook took control of acquiring a stats tuple.  If it did
                  * supply a tuple, it'd better have supplied a freefunc.
                  */
                 if (HeapTupleIsValid(vardata.statsTuple) &&
                     !vardata.freefunc)
                     elog(ERROR, "no function provided to release variable stats with");
             }
             else
             {
                 vardata.statsTuple = SearchSysCache3(STATRELATTINH,
                                                      ObjectIdGetDatum(relid),
                                                      Int16GetDatum(colnum),
                                                      BoolGetDatum(rte->inh));
                 vardata.freefunc = ReleaseSysCache;
             }
         }
         else
         {
             /* Expression --- maybe there are stats for the index itself */
             relid = index->indexoid;
             colnum = 1;
     
             if (get_index_stats_hook &&
                 (*get_index_stats_hook) (root, relid, colnum, &vardata))
             {
                 /*
                  * The hook took control of acquiring a stats tuple.  If it did
                  * supply a tuple, it'd better have supplied a freefunc.
                  */
                 if (HeapTupleIsValid(vardata.statsTuple) &&
                     !vardata.freefunc)
                     elog(ERROR, "no function provided to release variable stats with");
             }
             else
             {
                 vardata.statsTuple = SearchSysCache3(STATRELATTINH,
                                                      ObjectIdGetDatum(relid),
                                                      Int16GetDatum(colnum),
                                                      BoolGetDatum(false));
                 vardata.freefunc = ReleaseSysCache;
             }
         }
     
         if (HeapTupleIsValid(vardata.statsTuple))
         {
             Oid         sortop;
             AttStatsSlot sslot;
     
             sortop = get_opfamily_member(index->opfamily[0],
                                          index->opcintype[0],
                                          index->opcintype[0],
                                          BTLessStrategyNumber);
             if (OidIsValid(sortop) &&
                 get_attstatsslot(&sslot, vardata.statsTuple,
                                  STATISTIC_KIND_CORRELATION, sortop,
                                  ATTSTATSSLOT_NUMBERS))
             {
                 double      varCorrelation;
     
                 Assert(sslot.nnumbers == 1);
                 varCorrelation = sslot.numbers[0];
     
                 if (index->reverse_sort[0])
                     varCorrelation = -varCorrelation;
     
                 if (index->ncolumns > 1)
                     costs.indexCorrelation = varCorrelation * 0.75;
                 else
                     costs.indexCorrelation = varCorrelation;
     
                 free_attstatsslot(&sslot);
             }
         }
     
         ReleaseVariableStats(vardata);
     
         *indexStartupCost = costs.indexStartupCost;
         *indexTotalCost = costs.indexTotalCost;
         *indexSelectivity = costs.indexSelectivity;
         *indexCorrelation = costs.indexCorrelation;
         *indexPages = costs.numIndexPages;
     }
    
    //------------------------- index_pages_fetched
    
     /*
      * index_pages_fetched
      *    Estimate the number of pages actually fetched after accounting for
      *    cache effects.
      *    估算在考虑缓存影响后实际获取的页面数量。
      * 
      * 估算方法是Mackert和Lohman提出的方法:
      *     "Index Scans Using a Finite LRU Buffer: A Validated I/O Model"
      * We use an approximation proposed by Mackert and Lohman, "Index Scans
      * Using a Finite LRU Buffer: A Validated I/O Model", ACM Transactions
      * on Database Systems, Vol. 14, No. 3, September 1989, Pages 401-424.
      * The Mackert and Lohman approximation is that the number of pages
      * fetched is
      *  PF =
      *      min(2TNs/(2T+Ns), T)            when T <= b
      *      2TNs/(2T+Ns)                    when T > b and Ns <= 2Tb/(2T-b)
      *      b + (Ns - 2Tb/(2T-b))*(T-b)/T   when T > b and Ns > 2Tb/(2T-b)
      * where
      *      T = # pages in table
      *      N = # tuples in table
      *      s = selectivity = fraction of table to be scanned
      *      b = # buffer pages available (we include kernel space here)
      *
      * We assume that effective_cache_size is the total number of buffer pages
      * available for the whole query, and pro-rate that space across all the
      * tables in the query and the index currently under consideration.  (This
      * ignores space needed for other indexes used by the query, but since we
      * don't know which indexes will get used, we can't estimate that very well;
      * and in any case counting all the tables may well be an overestimate, since
      * depending on the join plan not all the tables may be scanned concurrently.)
      *
      * The product Ns is the number of tuples fetched; we pass in that
      * product rather than calculating it here.  "pages" is the number of pages
      * in the object under consideration (either an index or a table).
      * "index_pages" is the amount to add to the total table space, which was
      * computed for us by query_planner.
      *
      * Caller is expected to have ensured that tuples_fetched is greater than zero
      * and rounded to integer (see clamp_row_est).  The result will likewise be
      * greater than zero and integral.
      */
     double
     index_pages_fetched(double tuples_fetched, BlockNumber pages,
                         double index_pages, PlannerInfo *root)
     {
         double      pages_fetched;
         double      total_pages;
         double      T,
                     b;
     
         /* T is # pages in table, but don't allow it to be zero */
         T = (pages > 1) ? (double) pages : 1.0;
     
         /* Compute number of pages assumed to be competing for cache space */
         total_pages = root->total_table_pages + index_pages;
         total_pages = Max(total_pages, 1.0);
         Assert(T <= total_pages);
     
         /* b is pro-rated share of effective_cache_size */
         b = (double) effective_cache_size * T / total_pages;
     
         /* force it positive and integral */
         if (b <= 1.0)
             b = 1.0;
         else
             b = ceil(b);
     
         /* This part is the Mackert and Lohman formula */
         if (T <= b)
         {
             pages_fetched =
                 (2.0 * T * tuples_fetched) / (2.0 * T + tuples_fetched);
             if (pages_fetched >= T)
                 pages_fetched = T;
             else
                 pages_fetched = ceil(pages_fetched);
         }
         else
         {
             double      lim;
     
             lim = (2.0 * T * b) / (2.0 * T - b);
             if (tuples_fetched <= lim)
             {
                 pages_fetched =
                     (2.0 * T * tuples_fetched) / (2.0 * T + tuples_fetched);
             }
             else
             {
                 pages_fetched =
                     b + (tuples_fetched - lim) * (T - b) / T;
             }
             pages_fetched = ceil(pages_fetched);
         }
         return pages_fetched;
     }
    

### 三、跟踪分析

测试脚本如下

    
    
    select a.*,b.grbh,b.je 
    from t_dwxx a,
        lateral (select t1.dwbh,t1.grbh,t2.je 
         from t_grxx t1 
              inner join t_jfxx t2 on t1.dwbh = a.dwbh and t1.grbh = t2.grbh) b
    where a.dwbh = '1001'
    order by b.dwbh;
    

启动gdb

    
    
    (gdb) b create_index_path
    Breakpoint 1 at 0x78f050: file pathnode.c, line 1037.
    (gdb) c
    Continuing.
    
    

主要考察t_grxx上的索引访问路径,即t_grxx.dwbh = '1001'(通过等价类产生并下推的限制条件)

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 1, create_index_path (root=0x2737d70, index=0x274be80, indexclauses=0x274f1f8, indexclausecols=0x274f248, 
        indexorderbys=0x0, indexorderbycols=0x0, pathkeys=0x0, indexscandir=ForwardScanDirection, indexonly=false, 
        required_outer=0x0, loop_count=1, partial_path=false) at pathnode.c:1037
    1037        IndexPath  *pathnode = makeNode(IndexPath);
    

索引信息:树高度为1/索引列1个/indexlist链表,元素为TargetEntry,相关信息为varno = 3, varattno =
1,索引访问方法成本估算使用的函数为btcostestimate

    
    
    (gdb) p *index
    $3 = {type = T_IndexOptInfo, indexoid = 16752, reltablespace = 0, rel = 0x274b870, pages = 276, tuples = 100000, 
      tree_height = 1, ncolumns = 1, nkeycolumns = 1, indexkeys = 0x274bf90, indexcollations = 0x274bfa8, opfamily = 0x274bfc0, 
      opcintype = 0x274bfd8, sortopfamily = 0x274bfc0, reverse_sort = 0x274c008, nulls_first = 0x274c020, 
      canreturn = 0x274bff0, relam = 403, indexprs = 0x0, indpred = 0x0, indextlist = 0x274c0f8, indrestrictinfo = 0x274dc58, 
      predOK = false, unique = false, immediate = true, hypothetical = false, amcanorderbyop = false, amoptionalkey = true, 
      amsearcharray = true, amsearchnulls = true, amhasgettuple = true, amhasgetbitmap = true, amcanparallel = true, 
      amcostestimate = 0x94f0ad <btcostestimate>}
    

执行各项赋值操作

    
    
    (gdb) n
    1038        RelOptInfo *rel = index->rel;
    (gdb) 
    1042        pathnode->path.pathtype = indexonly ? T_IndexOnlyScan : T_IndexScan;
    (gdb) 
    1043        pathnode->path.parent = rel;
    (gdb) 
    1044        pathnode->path.pathtarget = rel->reltarget;
    (gdb) 
    1045        pathnode->path.param_info = get_baserel_parampathinfo(root, rel,
    (gdb) 
    1047        pathnode->path.parallel_aware = false;
    (gdb) 
    1048        pathnode->path.parallel_safe = rel->consider_parallel;
    (gdb) 
    1049        pathnode->path.parallel_workers = 0;
    (gdb) 
    1050        pathnode->path.pathkeys = pathkeys;
    (gdb) 
    1053        expand_indexqual_conditions(index, indexclauses, indexclausecols,
    (gdb) 
    1057        pathnode->indexinfo = index;
    

执行expand_indexqual_conditions,给定RestrictInfo节点(约束条件),产生直接可用的索引表达式子句

    
    
    (gdb) p *indexclauses
    $4 = {type = T_List, length = 1, head = 0x274f1d8, tail = 0x274f1d8} -->t_grxx.dwbh = '1001'
    (gdb) p *indexclausecols
    $9 = {type = T_IntList, length = 1, head = 0x274f228, tail = 0x274f228}
    (gdb) p indexclausecols->head->data.int_value
    $10 = 0
    

进入cost_index函数

    
    
    (gdb) 
    1065        cost_index(pathnode, root, loop_count, partial_path);
    (gdb) step
    cost_index (path=0x274ecb8, root=0x2737d70, loop_count=1, partial_path=false) at costsize.c:480
    480     IndexOptInfo *index = path->indexinfo;
    

调用访问方法成本估算函数

    
    
    ...
    (gdb) 
    547     amcostestimate(root, path, loop_count,
    (gdb) 
    557     path->indextotalcost = indexTotalCost;
    

相关返回值

    
    
    (gdb) p indexStartupCost
    $22 = 0.29249999999999998
    (gdb) p indexTotalCost
    $23 = 4.3675000000000006
    (gdb) p indexSelectivity
    $24 = 0.00010012021638664612
    (gdb) p indexCorrelation
    $25 = 0.82452213764190674
    (gdb) p index_pages
    $26 = 1
    

loop_count=1

    
    
    599     if (loop_count > 1)
    (gdb) 
    651                                             (double) index->pages,
    (gdb) p loop_count
    $27 = 1
    

取得的page数量,计算IO大小等

    
    
    (gdb) n
    649         pages_fetched = index_pages_fetched(tuples_fetched,
    (gdb) 
    654         if (indexonly)
    (gdb) p pages_fetched
    $28 = 10
    ...
    (gdb) p max_IO_cost
    $30 = 40
    (gdb) p min_IO_cost
    $31 = 4
    

调用完成,查看最终结果

    
    
    749     path->path.total_cost = startup_cost + run_cost;
    (gdb) 
    750 }
    (gdb) p *path
    $37 = {path = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x274b870, pathtarget = 0x274ba98, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.29249999999999998, 
        total_cost = 19.993376803383146, pathkeys = 0x0}, indexinfo = 0x274be80, indexclauses = 0x274f1f8, 
      indexquals = 0x274f3a0, indexqualcols = 0x274f3f0, indexorderbys = 0x0, indexorderbycols = 0x0, 
      indexscandir = ForwardScanDirection, indextotalcost = 4.3675000000000006, indexselectivity = 0.00010012021638664612}
    (gdb) n
    create_index_path (root=0x2737d70, index=0x274be80, indexclauses=0x274f1f8, indexclausecols=0x274f248, indexorderbys=0x0, 
        indexorderbycols=0x0, pathkeys=0x0, indexscandir=ForwardScanDirection, indexonly=false, required_outer=0x0, 
        loop_count=1, partial_path=false) at pathnode.c:1067
    1067        return pathnode;  
    

该SQL语句的执行计划,其中Index Scan using idx_t_grxx_dwbh on public.t_grxx t1
(cost=0.29..19.99...的成本0.29/19.99,与访问路径中的startup_cost/total_cost相对应.

    
    
    testdb=# explain verbose select a.*,b.grbh,b.je 
    from t_dwxx a,
        lateral (select t1.dwbh,t1.grbh,t2.je 
         from t_grxx t1 
              inner join t_jfxx t2 on t1.dwbh = a.dwbh and t1.grbh = t2.grbh) b
    where a.dwbh = '1001'
    order by b.dwbh;
                                                  QUERY PLAN                                              
    ------------------------------------------------------------------------------------------------------     Nested Loop  (cost=0.87..111.60 rows=10 width=37)
       Output: a.dwmc, a.dwbh, a.dwdz, t1.grbh, t2.je, t1.dwbh
       ->  Nested Loop  (cost=0.58..28.40 rows=10 width=29)
             Output: a.dwmc, a.dwbh, a.dwdz, t1.grbh, t1.dwbh
             ->  Index Scan using t_dwxx_pkey on public.t_dwxx a  (cost=0.29..8.30 rows=1 width=20)
                   Output: a.dwmc, a.dwbh, a.dwdz
                   Index Cond: ((a.dwbh)::text = '1001'::text)
             ->  Index Scan using idx_t_grxx_dwbh on public.t_grxx t1  (cost=0.29..19.99 rows=10 width=9)
                   Output: t1.dwbh, t1.grbh, t1.xm, t1.xb, t1.nl
                   Index Cond: ((t1.dwbh)::text = '1001'::text)
       ->  Index Scan using idx_t_jfxx_grbh on public.t_jfxx t2  (cost=0.29..8.31 rows=1 width=13)
             Output: t2.grbh, t2.ny, t2.je
             Index Cond: ((t2.grbh)::text = (t1.grbh)::text)
    (13 rows)
    

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

