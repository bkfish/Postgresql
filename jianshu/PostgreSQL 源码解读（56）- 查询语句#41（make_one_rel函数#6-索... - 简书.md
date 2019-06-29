这一小节主要介绍索引扫描成本估算中get_index_paths函数的主逻辑及其子函数build_index_paths,下一小节介绍子函数build_index_paths中的create_index_path。

### 一、数据结构

**IndexClauseSet**  
用于收集匹配索引的的条件语句

    
    
     /* Data structure for collecting qual clauses that match an index */
     typedef struct
     {
         bool        nonempty;       /* True if lists are not all empty */
         /* Lists of RestrictInfos, one per index column */
         List       *indexclauses[INDEX_MAX_KEYS];
     } IndexClauseSet;
    

### 二、源码解读

**get_index_paths**  
get_index_paths函数根据给定的索引和索引条件子句,构造索引访问路径(IndexPath).

    
    
    //--------------------------------------------------- get_index_paths
    
     /*
      * get_index_paths
      *    Given an index and a set of index clauses for it, construct IndexPaths.
      *    给定索引和索引条件子句,构造索引访问路径(IndexPaths).
      *
      * Plain indexpaths are sent directly to add_path, while potential
      * bitmap indexpaths are added to *bitindexpaths for later processing.
      * Plain索引访问路径直接作为参数传入到函数add_path中,潜在可能的位图索引路径
      * 被添加到bitindexpaths中供以后处理。
      *
      * This is a fairly simple frontend to build_index_paths().  Its reason for
      * existence is mainly to handle ScalarArrayOpExpr quals properly.  If the
      * index AM supports them natively, we should just include them in simple
      * index paths.  If not, we should exclude them while building simple index
      * paths, and then make a separate attempt to include them in bitmap paths.
      * Furthermore, we should consider excluding lower-order ScalarArrayOpExpr
      * quals so as to create ordered paths.
      * 该函数简单构造build_index_paths所需要的参数,并调用之.该函数存在的原因是正确
      * 处理ScalarArrayOpExpr表达式.
      */
     static void
     get_index_paths(PlannerInfo *root, RelOptInfo *rel,
                     IndexOptInfo *index, IndexClauseSet *clauses,
                     List **bitindexpaths)
     {
         List       *indexpaths;
         bool        skip_nonnative_saop = false;
         bool        skip_lower_saop = false;
         ListCell   *lc;
     
         /*
          * Build simple index paths using the clauses.  Allow ScalarArrayOpExpr
          * clauses only if the index AM supports them natively, and skip any such
          * clauses for index columns after the first (so that we produce ordered
          * paths if possible).
          */
         indexpaths = build_index_paths(root, rel,
                                        index, clauses,
                                        index->predOK,
                                        ST_ANYSCAN,
                                        &skip_nonnative_saop,
                                        &skip_lower_saop);//调用build_index_paths函数
     
         /*
          * If we skipped any lower-order ScalarArrayOpExprs on an index with an AM
          * that supports them, then try again including those clauses.  This will
          * produce paths with more selectivity but no ordering.
          */
         if (skip_lower_saop)
         {
             indexpaths = list_concat(indexpaths,
                                      build_index_paths(root, rel,
                                                        index, clauses,
                                                        index->predOK,
                                                        ST_ANYSCAN,
                                                        &skip_nonnative_saop,
                                                        NULL));
         }
     
         /*
          * Submit all the ones that can form plain IndexScan plans to add_path. (A
          * plain IndexPath can represent either a plain IndexScan or an
          * IndexOnlyScan, but for our purposes here that distinction does not
          * matter.  However, some of the indexes might support only bitmap scans,
          * and those we mustn't submit to add_path here.)
          * 逐一把可以形成Plain索引扫描计划的的访问路径作为参数调用add_path
          * (plain IndexPath可以是常规的索引扫描或者是IndexOnlyScan)
          *
          * Also, pick out the ones that are usable as bitmap scans.  For that, we
          * must discard indexes that don't support bitmap scans, and we also are
          * only interested in paths that have some selectivity; we should discard
          * anything that was generated solely for ordering purposes.
          * 找出可用于位图扫描的索引
          */
         foreach(lc, indexpaths)//遍历访问路径
         {
             IndexPath  *ipath = (IndexPath *) lfirst(lc);
     
             if (index->amhasgettuple)
                 add_path(rel, (Path *) ipath);//调用add_path
     
             if (index->amhasgetbitmap &&
                 (ipath->path.pathkeys == NIL ||
                  ipath->indexselectivity < 1.0))
                 *bitindexpaths = lappend(*bitindexpaths, ipath);//如可以,添加到位图索引扫描路径链表中
         }
     
         /*
          * If there were ScalarArrayOpExpr clauses that the index can't handle
          * natively, generate bitmap scan paths relying on executor-managed
          * ScalarArrayOpExpr.
          */
         if (skip_nonnative_saop)
         {
             indexpaths = build_index_paths(root, rel,
                                            index, clauses,
                                            false,
                                            ST_BITMAPSCAN,
                                            NULL,
                                            NULL);
             *bitindexpaths = list_concat(*bitindexpaths, indexpaths);
         }
     }
     
    //----------------------------------------------------------- build_index_paths
     /*
      * build_index_paths
      *    Given an index and a set of index clauses for it, construct zero
      *    or more IndexPaths. It also constructs zero or more partial IndexPaths.
      *    给定索引和该索引的条件,构造0-N个索引访问路径或partial索引访问路径(用于并行)
      *
      * We return a list of paths because (1) this routine checks some cases
      * that should cause us to not generate any IndexPath, and (2) in some
      * cases we want to consider both a forward and a backward scan, so as
      * to obtain both sort orders.  Note that the paths are just returned
      * to the caller and not immediately fed to add_path().
      * 函数返回访问路径链表:(1)执行过程中的检查会导致产生不了索引访问路径 
      * (2)在某些情况下，同时考虑正向/反向扫描，以便获得两种排序顺序。
      * 注意:访问路径返回给调用方,不会马上反馈到函数add_path中
      *
      * At top level, useful_predicate should be exactly the index's predOK flag
      * (ie, true if it has a predicate that was proven from the restriction
      * clauses).  When working on an arm of an OR clause, useful_predicate
      * should be true if the predicate required the current OR list to be proven.
      * Note that this routine should never be called at all if the index has an
      * unprovable predicate.
      * 在顶层,useful_predicate标记应与索引的predOK标记一致.在操作OR自己的arm(?)时,
      * 如果谓词需要当前的OR链表证明,则useful_predicate应为T.
      * 注意:如果索引有一个未经验证的谓词,则该例程不能被调用.
      *
      * scantype indicates whether we want to create plain indexscans, bitmap
      * indexscans, or both.  When it's ST_BITMAPSCAN, we will not consider
      * index ordering while deciding if a Path is worth generating.
      * scantype:提示是否创建plain/bitmap或者两者兼顾的索引扫描.
      * 如该参数值为ST_BITMAPSCAN,则在决定访问路径是否产生时不会考虑使用索引排序
      *
      * If skip_nonnative_saop is non-NULL, we ignore ScalarArrayOpExpr clauses
      * unless the index AM supports them directly, and we set *skip_nonnative_saop
      * to true if we found any such clauses (caller must initialize the variable
      * to false).  If it's NULL, we do not ignore ScalarArrayOpExpr clauses.
      * skip_nonnative_saop:
      * 如为NOT NULL,除非索引的访问方法(AM)直接支持,否则会忽略
      *     ScalarArrayOpExpr子句,如支持,则更新skip_nonnative_saop标记为T.
      * 如为NULL,不能忽略ScalarArrayOpExpr子句.
      *
      * If skip_lower_saop is non-NULL, we ignore ScalarArrayOpExpr clauses for
      * non-first index columns, and we set *skip_lower_saop to true if we found
      * any such clauses (caller must initialize the variable to false).  If it's
      * NULL, we do not ignore non-first ScalarArrayOpExpr clauses, but they will
      * result in considering the scan's output to be unordered.
      * skip_lower_saop:
      * 如为NOT NULL,ScalarArrayOpExpr子句中的首列不是索引列,则忽略之,
      *     同时如果找到相应的子句,则设置skip_lower_saop标记为T.
      * 如为NULL,除首个ScalarArrayOpExpr子句外,其他子句不能被忽略,但输出时不作排序
      * 
      * 输入/输出参数:
      * 'rel' is the index's heap relation
      * rel-相应的RelOptInfo
      * 'index' is the index for which we want to generate paths
      * index-相应的索引IndexOptInfo
      * 'clauses' is the collection of indexable clauses (RestrictInfo nodes)
      * clauses-RestrictInfo节点集合
      * 'useful_predicate' indicates whether the index has a useful predicate
      * useful_predicate-提示索引是否有可用的谓词
      * 'scantype' indicates whether we need plain or bitmap scan support
      * scantype-扫描类型,提示是否需要plain/bitmap扫描支持
      * 'skip_nonnative_saop' indicates whether to accept SAOP if index AM doesn't
      * skip_nonnative_saop-提示是否接受SAOP
      * 'skip_lower_saop' indicates whether to accept non-first-column SAOP
      * skip_lower_saop-提示是否接受非首列SAOP
      */
     static List *
     build_index_paths(PlannerInfo *root, RelOptInfo *rel,
                       IndexOptInfo *index, IndexClauseSet *clauses,
                       bool useful_predicate,
                       ScanTypeControl scantype,
                       bool *skip_nonnative_saop,
                       bool *skip_lower_saop)
     {
         List       *result = NIL;
         IndexPath  *ipath;
         List       *index_clauses;
         List       *clause_columns;
         Relids      outer_relids;
         double      loop_count;
         List       *orderbyclauses;
         List       *orderbyclausecols;
         List       *index_pathkeys;
         List       *useful_pathkeys;
         bool        found_lower_saop_clause;
         bool        pathkeys_possibly_useful;
         bool        index_is_ordered;
         bool        index_only_scan;
         int         indexcol;
     
         /*
          * Check that index supports the desired scan type(s)
          */
         switch (scantype)
         {
             case ST_INDEXSCAN:
                 if (!index->amhasgettuple)
                     return NIL;
                 break;
             case ST_BITMAPSCAN:
                 if (!index->amhasgetbitmap)
                     return NIL;
                 break;
             case ST_ANYSCAN:
                 /* either or both are OK */
                 break;
         }
     
         /*
          * 1. Collect the index clauses into a single list.
          * 1. 收集索引子句到单独的链表中 
          *
          * We build a list of RestrictInfo nodes for clauses to be used with this
          * index, along with an integer list of the index column numbers (zero
          * based) that each clause should be used with.  The clauses are ordered
          * by index key, so that the column numbers form a nondecreasing sequence.
          * (This order is depended on by btree and possibly other places.)  The
          * lists can be empty, if the index AM allows that.
          * 我们为将与此索引一起使用的子句构建了一个RestrictInfo节点链表，
          * 以及每个子句应该与之一起使用的索引列编号(从0开始)的整数链表.
          * 子句是按索引键排序的，因此列号形成一个非递减序列。
          *  (这个排序取决于BTree和可能的其他地方).
          * 如果索引访问方法(AM)允许,链表可为空.
          *
          * found_lower_saop_clause is set true if we accept a ScalarArrayOpExpr
          * index clause for a non-first index column.  This prevents us from
          * assuming that the scan result is ordered.  (Actually, the result is
          * still ordered if there are equality constraints for all earlier
          * columns, but it seems too expensive and non-modular for this code to be
          * aware of that refinement.)
          *
          * We also build a Relids set showing which outer rels are required by the
          * selected clauses.  Any lateral_relids are included in that, but not
          * otherwise accounted for.
          * 建立一个已选择的子句所依赖外部rels的Relids集合,包括lateral_relids.
          */
         index_clauses = NIL;
         clause_columns = NIL;
         found_lower_saop_clause = false;
         outer_relids = bms_copy(rel->lateral_relids);
         for (indexcol = 0; indexcol < index->ncolumns; indexcol++)//遍历索引列
         {
             ListCell   *lc;
     
             foreach(lc, clauses->indexclauses[indexcol])//遍历该列所对应的约束条件
             {
                 RestrictInfo *rinfo = (RestrictInfo *) lfirst(lc);//约束条件
     
                 if (IsA(rinfo->clause, ScalarArrayOpExpr))//ScalarArrayOpExpr,TODO
                 {
                     if (!index->amsearcharray)
                     {
                         if (skip_nonnative_saop)
                         {
                             /* Ignore because not supported by index */
                             *skip_nonnative_saop = true;
                             continue;
                         }
                         /* Caller had better intend this only for bitmap scan */
                         Assert(scantype == ST_BITMAPSCAN);
                     }
                     if (indexcol > 0)
                     {
                         if (skip_lower_saop)
                         {
                             /* Caller doesn't want to lose index ordering */
                             *skip_lower_saop = true;
                             continue;
                         }
                         found_lower_saop_clause = true;
                     }
                 }
                 index_clauses = lappend(index_clauses, rinfo);//添加到链表中
                 clause_columns = lappend_int(clause_columns, indexcol);
                 outer_relids = bms_add_members(outer_relids,
                                                rinfo->clause_relids);
             }
     
             /*
              * If no clauses match the first index column, check for amoptionalkey
              * restriction.  We can't generate a scan over an index with
              * amoptionalkey = false unless there's at least one index clause.
              * (When working on columns after the first, this test cannot fail. It
              * is always okay for columns after the first to not have any
              * clauses.)
              */
             if (index_clauses == NIL && !index->amoptionalkey)//没有约束条件,返回空指针
                 return NIL;
         }
     
         /* We do not want the index's rel itself listed in outer_relids */
         outer_relids = bms_del_member(outer_relids, rel->relid);//去掉自身relid
         /* Enforce convention that outer_relids is exactly NULL if empty */
         if (bms_is_empty(outer_relids))
             outer_relids = NULL;//设置为NULL
     
         /* Compute loop_count for cost estimation purposes */
         //计算成本估算所需要的循环次数loop_count
         loop_count = get_loop_count(root, rel->relid, outer_relids);
     
         /*
          * 2. Compute pathkeys describing index's ordering, if any, then see how
          * many of them are actually useful for this query.  This is not relevant
          * if we are only trying to build bitmap indexscans, nor if we have to
          * assume the scan is unordered.
          * 2.计算描述索引排序的路径键(如果有的话)，如存在的话,检查有多少对查询有用。
          *   如果只是尝试构建位图索引扫描，或者扫描是无序的，那就无关紧要了。
          */
         pathkeys_possibly_useful = (scantype != ST_BITMAPSCAN &&
                                     !found_lower_saop_clause &&
                                     has_useful_pathkeys(root, rel));//是否存在可用的Pathkeys
         index_is_ordered = (index->sortopfamily != NULL);//索引是否排序?
         if (index_is_ordered && pathkeys_possibly_useful)//索引排序&存在可用的Pathkeys
         {
             index_pathkeys = build_index_pathkeys(root, index,
                                                   ForwardScanDirection);//创建正向(ForwardScanDirection)扫描排序键
             useful_pathkeys = truncate_useless_pathkeys(root, rel,
                                                         index_pathkeys);//截断无用的Pathkeys
             orderbyclauses = NIL;
             orderbyclausecols = NIL;
         }
         else if (index->amcanorderbyop && pathkeys_possibly_useful)//访问方法可以通过操作符排序
         {
             /* see if we can generate ordering operators for query_pathkeys */
             match_pathkeys_to_index(index, root->query_pathkeys,
                                     &orderbyclauses,
                                     &orderbyclausecols);//是否可以生成排序操作符
             if (orderbyclauses)
                 useful_pathkeys = root->query_pathkeys;//如可以,则赋值
             else
                 useful_pathkeys = NIL;
         }
         else//设置为NULL
         {
             useful_pathkeys = NIL;
             orderbyclauses = NIL;
             orderbyclausecols = NIL;
         }
     
         /*
          * 3. Check if an index-only scan is possible.  If we're not building
          * plain indexscans, this isn't relevant since bitmap scans don't support
          * index data retrieval anyway.
          * 3. 检查是否只需要扫描索引.如果我们不构建纯索引扫描，这是无关紧要的,
         *     因为位图扫描不支持索引数据检索。
          */
         index_only_scan = (scantype != ST_BITMAPSCAN &&
                            check_index_only(rel, index));
     
         /*
          * 4. Generate an indexscan path if there are relevant restriction clauses
          * in the current clauses, OR the index ordering is potentially useful for
          * later merging or final output ordering, OR the index has a useful
          * predicate, OR an index-only scan is possible.
          * 4. 如果当前子句中有相关的限制子句，或者索引排序对于以后的
          *    合并或最终的输出排序可能有用，或者索引存在可用的谓词，
          *    或者可能进行纯索引扫描，则生成索引扫描路径。
          */
         if (index_clauses != NIL || useful_pathkeys != NIL || useful_predicate ||
             index_only_scan)
         {
             ipath = create_index_path(root, index,
                                       index_clauses,
                                       clause_columns,
                                       orderbyclauses,
                                       orderbyclausecols,
                                       useful_pathkeys,
                                       index_is_ordered ?
                                       ForwardScanDirection :
                                       NoMovementScanDirection,
                                       index_only_scan,
                                       outer_relids,
                                       loop_count,
                                       false);//创建索引扫描路径
             result = lappend(result, ipath);//添加到结果链表中
     
             /*
              * If appropriate, consider parallel index scan.  We don't allow
              * parallel index scan for bitmap index scans.
              * 如果合适，考虑并行索引扫描。如为位图索引,则不能使用并行.。
              */
             if (index->amcanparallel &&
                 rel->consider_parallel && outer_relids == NULL &&
                 scantype != ST_BITMAPSCAN)
             {
                 ipath = create_index_path(root, index,
                                           index_clauses,
                                           clause_columns,
                                           orderbyclauses,
                                           orderbyclausecols,
                                           useful_pathkeys,
                                           index_is_ordered ?
                                           ForwardScanDirection :
                                           NoMovementScanDirection,
                                           index_only_scan,
                                           outer_relids,
                                           loop_count,
                                           true);//创建并行索引扫描路径
     
                 /*
                  * if, after costing the path, we find that it's not worth using
                  * parallel workers, just free it.
                  */
                 if (ipath->path.parallel_workers > 0)//在worker>0的情况下
                     add_partial_path(rel, (Path *) ipath);//添加partial路径
                 else
                     pfree(ipath);
             }
         }
     
         /*
          * 5. If the index is ordered, a backwards scan might be interesting.
          * 5. 如果索引是已排序的(如BTree等),构建反向扫描(BackwardScanDirection)路径
          */
         if (index_is_ordered && pathkeys_possibly_useful)
         {
             index_pathkeys = build_index_pathkeys(root, index,
                                                   BackwardScanDirection);
             useful_pathkeys = truncate_useless_pathkeys(root, rel,
                                                         index_pathkeys);
             if (useful_pathkeys != NIL)
             {
                 ipath = create_index_path(root, index,
                                           index_clauses,
                                           clause_columns,
                                           NIL,
                                           NIL,
                                           useful_pathkeys,
                                           BackwardScanDirection,
                                           index_only_scan,
                                           outer_relids,
                                           loop_count,
                                           false);
                 result = lappend(result, ipath);
     
                 /* If appropriate, consider parallel index scan */
                 if (index->amcanparallel &&
                     rel->consider_parallel && outer_relids == NULL &&
                     scantype != ST_BITMAPSCAN)
                 {
                     ipath = create_index_path(root, index,
                                               index_clauses,
                                               clause_columns,
                                               NIL,
                                               NIL,
                                               useful_pathkeys,
                                               BackwardScanDirection,
                                               index_only_scan,
                                               outer_relids,
                                               loop_count,
                                               true);
     
                     /*
                      * if, after costing the path, we find that it's not worth
                      * using parallel workers, just free it.
                      */
                     if (ipath->path.parallel_workers > 0)
                         add_partial_path(rel, (Path *) ipath);
                     else
                         pfree(ipath);
                 }
             }
         }
     
         return result;
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

    
    
    (gdb) b get_index_paths
    Breakpoint 1 at 0x74db5e: file indxpath.c, line 740.
    (gdb) c
    Continuing.
    
    Breakpoint 1, get_index_paths (root=0x2704720, rel=0x27041b0, index=0x2713898, clauses=0x7fff834d8090, 
        bitindexpaths=0x7fff834d81b0) at indxpath.c:740
    740   bool    skip_nonnative_saop = false;
    

选择t_dwxx的主键进行跟踪

    
    
    (gdb) p *index
    $2 = {type = T_IndexOptInfo, indexoid = 16738, reltablespace = 0, rel = 0x27041b0, pages = 30, tuples = 10000, 
      tree_height = 1, ncolumns = 1, nkeycolumns = 1, indexkeys = 0x2713bd8, indexcollations = 0x2713bf0, opfamily = 0x2713c08, 
      opcintype = 0x2713c20, sortopfamily = 0x2713c08, reverse_sort = 0x2713c50, nulls_first = 0x2713c68, 
      canreturn = 0x2713c38, relam = 403, indexprs = 0x0, indpred = 0x0, indextlist = 0x2713ba8, indrestrictinfo = 0x2716c18, 
      predOK = false, unique = true, immediate = true, hypothetical = false, amcanorderbyop = false, amoptionalkey = true, 
      amsearcharray = true, amsearchnulls = true, amhasgettuple = true, amhasgetbitmap = true, amcanparallel = true, 
      amcostestimate = 0x94f0ad <btcostestimate>}
    --    testdb=# select relname from pg_class where oid=16738;
       relname   
    -------------     t_dwxx_pkey
    (1 row)
    --  
    

进入函数build_index_paths

    
    
    (gdb) step
    build_index_paths (root=0x2704720, rel=0x27041b0, index=0x27135b8, clauses=0x7fff834d8090, useful_predicate=false, 
        scantype=ST_ANYSCAN, skip_nonnative_saop=0x7fff834d7e27, skip_lower_saop=0x7fff834d7e26) at indxpath.c:864
    864   List     *result = NIL;
    

查看输入参数,其中clauses中的indexclauses数组,存储约束条件链表

    
    
    (gdb) p *clauses
    $3 = {nonempty = true, indexclauses = {0x2717728, 0x0 <repeats 31 times>}}
    (gdb) set $ri=(RestrictInfo *)clauses->indexclauses[0]->head->data.ptr_value
    

链表的第一个约束条件为:t_dwxx.dwbh(varno = 1, varattno = 2)='1001'(constvalue = 40986784)

    
    
    (gdb) p *((OpExpr *)$ri->clause)->args
    $11 = {type = T_List, length = 2, head = 0x27169f8, tail = 0x27169a8}
    (gdb) p *(Node *)((OpExpr *)$ri->clause)->args->head->data.ptr_value
    $12 = {type = T_RelabelType}
    (gdb) p *(Node *)((OpExpr *)$ri->clause)->args->head->next->data.ptr_value
    $13 = {type = T_Const}
    (gdb) set $tmp1=(RelabelType *)((OpExpr *)$ri->clause)->args->head->data.ptr_value
    (gdb) set $tmp2=(Const *)((OpExpr *)$ri->clause)->args->head->next->data.ptr_value
    (gdb) p *(Var *)$tmp1->arg
    $17 = {xpr = {type = T_Var}, varno = 1, varattno = 2, vartype = 1043, vartypmod = 24, varcollid = 100, varlevelsup = 0, 
      varnoold = 1, varoattno = 2, location = 147}
    (gdb) p *(Const *)$tmp2
    $18 = {xpr = {type = T_Const}, consttype = 25, consttypmod = -1, constcollid = 100, constlen = -1, constvalue = 40986784, 
      constisnull = false, constbyval = false, location = 194}
    

扫描类型,ST_ANYSCAN,包括plain&bitmap

    
    
    (gdb) n
    883   switch (scantype)
    (gdb) p scantype
    $19 = ST_ANYSCAN
    

Step 1:收集索引子句到单独的链表中

    
    
    923   for (indexcol = 0; indexcol < index->ncolumns; indexcol++)
    (gdb) p outer_relids
    $20 = (Relids) 0x0
    (gdb) n
    927     foreach(lc, clauses->indexclauses[indexcol])
    (gdb) p indexcol
    $21 = 0
    (gdb) n
    #rinfo约束条件:t_dwxx.dwbh='1001'
    929       RestrictInfo *rinfo = (RestrictInfo *) lfirst(lc);
    (gdb) 
    931       if (IsA(rinfo->clause, ScalarArrayOpExpr))
    (gdb) 
    

Step 1的主要逻辑:

    
    
    (gdb) n
    955       index_clauses = lappend(index_clauses, rinfo);
    (gdb) p *index_clauses
    Cannot access memory at address 0x0
    (gdb) n
    956       clause_columns = lappend_int(clause_columns, indexcol);
    (gdb) 
    958                        rinfo->clause_relids);
    (gdb) 
    957       outer_relids = bms_add_members(outer_relids,
    

Step 1完成后:

    
    
    (gdb) p *outer_relids
    $23 = {nwords = 1, words = 0x27177fc}
    (gdb) p *index_clauses
    $26 = {type = T_List, length = 1, head = 0x2717758, tail = 0x2717758}
    (gdb) p outer_relids->words[0]
    $27 = 0  -->无外部的Relids
    (gdb) p *clause_columns
    $31 = {type = T_IntList, length = 1, head = 0x27177a8, tail = 0x27177a8}
    (gdb) p clause_columns->head->data.int_value
    $32 = 0 -->列数组编号为0
    

循环次数:

    
    
    (gdb) p loop_count
    $33 = 1
    

Step 2:计算描述索引排序的路径键(如果有的话)，如存在的话,检查有多少对查询有用。

    
    
    ...
    (gdb) p pathkeys_possibly_useful
    $35 = true
    (gdb) p index_is_ordered
    $36 = true
    

创建正向扫描排序键

    
    
    (gdb) n
    994     index_pathkeys = build_index_pathkeys(root, index,
    (gdb) p *index_pathkeys
    Cannot access memory at address 0x0 -->无需排序
    

Step 3:检查是否只需要扫描索引

    
    
    (gdb) p index_only_scan
    $37 = false -->No way!
    

Step 4:生成索引扫描路径  
调用函数create_index_path(下节介绍)

    
    
    1036      ipath = create_index_path(root, index,
    (gdb) 
    1049      result = lappend(result, ipath);
    (gdb) p *ipath
    $38 = {path = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x27041b0, pathtarget = 0x27134c8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 1, startup_cost = 0.28500000000000003, 
        total_cost = 8.3025000000000002, pathkeys = 0x0}, indexinfo = 0x27135b8, indexclauses = 0x2717778, 
      indexquals = 0x27178e8, indexqualcols = 0x2717938, indexorderbys = 0x0, indexorderbycols = 0x0, 
      indexscandir = ForwardScanDirection, indextotalcost = 4.2925000000000004, indexselectivity = 0.0001}
    

Step 5:构建反向扫描(BackwardScanDirection)路径

    
    
    (gdb) p index_is_ordered
    $41 = true
    (gdb) p pathkeys_possibly_useful
    $42 = true
    ...
    (gdb) p *index_pathkeys
    Cannot access memory at address 0x0 -->无需排序
    

返回结果

    
    
    1137    return result;
    (gdb) 
    1138  }
    

至此函数调用结束.

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

