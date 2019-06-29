这一小节继续介绍查询物理优化中的create_index_paths->choose_bitmap_and,该函数执行Bitmap
AND操作后创建位图索引扫描访问路径(BitmapAndPath)节点。  
关于Bitmap Scan的相关知识,请参照[PostgreSQL DBA(6) - SeqScan vs IndexScan vs
BitmapHeapScan](https://www.jianshu.com/p/4f819a43882d)这篇文章.

下面是BitmapAnd访问路径的样例:

    
    
    testdb=# explain verbose select t1.* 
    testdb-# from t_dwxx t1 
    testdb-# where (dwbh > '10000' and dwbh < '15000') AND (dwdz between 'DWDZ10000' and 'DWDZ15000');
    QUERY PLAN                                
                                                        
    ----------------------------------------------------------------------------------------------     Bitmap Heap Scan on public.t_dwxx t1  (cost=32.33..88.38 rows=33 width=20)
       Output: dwmc, dwbh, dwdz
       Recheck Cond: (((t1.dwbh)::text > '10000'::text) AND ((t1.dwbh)::text < '15000'::text) AND ((t1.dwdz)::text >= 'DWDZ10000'
    ::text) AND ((t1.dwdz)::text <= 'DWDZ15000'::text))
       ->  BitmapAnd  (cost=32.33..32.33 rows=33 width=0)  -->BitmapAnd
             ->  Bitmap Index Scan on t_dwxx_pkey  (cost=0.00..13.86 rows=557 width=0)
                   Index Cond: (((t1.dwbh)::text > '10000'::text) AND ((t1.dwbh)::text < '15000'::text))
             ->  Bitmap Index Scan on idx_dwxx_dwdz  (cost=0.00..18.21 rows=592 width=0)
                   Index Cond: (((t1.dwdz)::text >= 'DWDZ10000'::text) AND ((t1.dwdz)::text <= 'DWDZ15000'::text))
    (8 rows)
    

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
    

**PathClauseUsage**

    
    
     /* Per-path data used within choose_bitmap_and() */
     typedef struct
     {
         Path       *path;           /* 访问路径链表,IndexPath, BitmapAndPath, or BitmapOrPath */
         List       *quals;          /* 限制条件子句链表,the WHERE clauses it uses */
         List       *preds;          /* 部分索引谓词链表,predicates of its partial index(es) */
         Bitmapset  *clauseids;      /* 位图集合,quals+preds represented as a bitmapset */
     } PathClauseUsage;
    

### 二、源码解读

**choose_bitmap_and函数**  
create_index_paths->choose_bitmap_and函数,该函数给定非空的位图访问路径链表,执行AND操作后合并到一条路径中,最终得到位图索引扫描访问路径节点.

    
    
     /*
      * choose_bitmap_and
      *      Given a nonempty list of bitmap paths, AND them into one path.
      *      给定非空的位图访问路径链表,执行AND操作后合并到一条路径中
      *
      * This is a nontrivial decision since we can legally use any subset of the
      * given path set.  We want to choose a good tradeoff between selectivity
      * and cost of computing the bitmap.
      * 这是一个非常重要的策略，因为这样可以合法地使用给定路径集的任何子集。
      * 
      * The result is either a single one of the inputs, or a BitmapAndPath
      * combining multiple inputs.
      * 输出结果要么是输出的其中之一,要么是融合多个输入之后的BitmapAndPath
      */
     static Path *
     choose_bitmap_and(PlannerInfo *root, RelOptInfo *rel, List *paths)
     {
         int         npaths = list_length(paths);
         PathClauseUsage **pathinfoarray;
         PathClauseUsage *pathinfo;
         List       *clauselist;
         List       *bestpaths = NIL;
         Cost        bestcost = 0;
         int         i,
                     j;
         ListCell   *l;
     
         Assert(npaths > 0);         /* else caller error */
         if (npaths == 1)
             return (Path *) linitial(paths);    /* easy case */
     
         /*
          * In theory we should consider every nonempty subset of the given paths.
          * In practice that seems like overkill, given the crude nature of the
          * estimates, not to mention the possible effects of higher-level AND and
          * OR clauses.  Moreover, it's completely impractical if there are a large
          * number of paths, since the work would grow as O(2^N).
          * 理论上，我们应该考虑给定路径的所有非空子集。在实践中，
          * 考虑到估算的不确定性和成本，以及更高级别的AND和OR约束可能产生的影响,这样的做法并不合适.
          * 此外,它并不切合实际,如果有大量的路径,这项工作的复杂度会是指数级的O(2 ^ N)。
          *
          * As a heuristic, we first check for paths using exactly the same sets of
          * WHERE clauses + index predicate conditions, and reject all but the
          * cheapest-to-scan in any such group.  This primarily gets rid of indexes
          * that include the interesting columns but also irrelevant columns.  (In
          * situations where the DBA has gone overboard on creating variant
          * indexes, this can make for a very large reduction in the number of
          * paths considered further.)
          * 作为一种启发式方法，首先使用完全相同的WHERE子句+索引谓词条件集检查路径，
          * 并去掉这类条件组中除成本最低之外的所有路径。
          * 这主要是去掉了包含interesting列和不相关列的索引。
          * (在DBA过度创建索引的情况下，这会大大减少进一步考虑的路径数量。)
          * 
          * We then sort the surviving paths with the cheapest-to-scan first, and
          * for each path, consider using that path alone as the basis for a bitmap
          * scan.  Then we consider bitmap AND scans formed from that path plus
          * each subsequent (higher-cost) path, adding on a subsequent path if it
          * results in a reduction in the estimated total scan cost. This means we
          * consider about O(N^2) rather than O(2^N) path combinations, which is
          * quite tolerable, especially given than N is usually reasonably small
          * because of the prefiltering step.  The cheapest of these is returned.
          * 然后，我们首先使用成本最低的扫描路径对现存的路径进行排序，
          * 对于每个路径，考虑单独使用该路径作为位图扫描的基础。
          * 然后我们考虑位图和从该路径形成的扫描加上每个后续的(更高成本的)路径，
          * 如果后续路径导致估算的总扫描成本减少，那么就添加一个后续路径。
          * 这意味着我们只需要处理O(N ^ 2),而不是O(2 ^ N)个路径组合,
          * 这样的成本完全可以接受,特别是N通常相当小时。函数返回成本最低的路径。
          * 
          * We will only consider AND combinations in which no two indexes use the
          * same WHERE clause.  This is a bit of a kluge: it's needed because
          * costsize.c and clausesel.c aren't very smart about redundant clauses.
          * They will usually double-count the redundant clauses, producing a
          * too-small selectivity that makes a redundant AND step look like it
          * reduces the total cost.  Perhaps someday that code will be smarter and
          * we can remove this limitation.  (But note that this also defends
          * against flat-out duplicate input paths, which can happen because
          * match_join_clauses_to_index will find the same OR join clauses that
          * extract_restriction_or_clauses has pulled OR restriction clauses out
          * of.)
          * 我们将只考虑没有两个索引同时使用相同的WHERE子句的AND组合。
          * 这是一个有点蹩脚的做法:之所以这样是因为cost.c和clausesel.c未能足够聪明的处理多余的子句。
          * 它们通常会重复计算冗余子句，从而产生很小的选择性，使冗余子句看起来像是减少了总成本。
          * 也许有一天，代码会变得更聪明，我们可以消除这个限制。
          * (但是要注意，这也可以防止完全重复的输入路径，
          * 因为match_join_clauses_to_index会找到相同的OR连接子句,而这些子句
          * 已通过extract_restriction_or_clauses函数提升到外面去了.)
          *
          * For the same reason, we reject AND combinations in which an index
          * predicate clause duplicates another clause.  Here we find it necessary
          * to be even stricter: we'll reject a partial index if any of its
          * predicate clauses are implied by the set of WHERE clauses and predicate
          * clauses used so far.  This covers cases such as a condition "x = 42"
          * used with a plain index, followed by a clauseless scan of a partial
          * index "WHERE x >= 40 AND x < 50".  The partial index has been accepted
          * only because "x = 42" was present, and so allowing it would partially
          * double-count selectivity.  (We could use predicate_implied_by on
          * regular qual clauses too, to have a more intelligent, but much more
          * expensive, check for redundancy --- but in most cases simple equality
          * seems to suffice.)
          * 出于同样的原因，我们不会组合索引谓词子句与另一个重复的子句。
          * 在这里，有必要更加严格 : 如果部分索引的任何谓词子句
          * 隐含在WHERE子句中，则不能使用此索引。
          * 这里包括了形如使用普通索引的“x = 42”和使用部分索引“x >= 40和x < 50”的情况。
          * 部分索引被接受，是因为存在“x = 42”，因此允许它部分重复计数选择性。
          * (我们也可以在普通的qual子句上使用predicate_implied_by函数，
          * 这样就可以更智能但更昂贵地检查冗余——但在大多数情况下，简单的等式似乎就足够了。)
          */
     
         /*
          * Extract clause usage info and detect any paths that use exactly the
          * same set of clauses; keep only the cheapest-to-scan of any such groups.
          * The surviving paths are put into an array for qsort'ing.
          * 提取子句使用信息并检测使用完全相同子句集的所有路径;
          * 只保留这类路径中成本最低的,这些路径被放入一个数组中进行qsort'ing
          */
         pathinfoarray = (PathClauseUsage **)
             palloc(npaths * sizeof(PathClauseUsage *));//数组
         clauselist = NIL;
         npaths = 0;
         foreach(l, paths)//遍历paths
         {
             Path       *ipath = (Path *) lfirst(l);
     
             pathinfo = classify_index_clause_usage(ipath, &clauselist);//归类路径信息
             for (i = 0; i < npaths; i++)
             {
                 if (bms_equal(pathinfo->clauseids, pathinfoarray[i]->clauseids))
                     break;//只要发现子句集一样,就继续执行
             }
             if (i < npaths)//发现相同的
             {
                 /* duplicate clauseids, keep the cheaper one */
                 //相同的约束条件,只保留成本最低的
                 Cost        ncost;
                 Cost        ocost;
                 Selectivity nselec;
                 Selectivity oselec;
     
                 cost_bitmap_tree_node(pathinfo->path, &ncost, &nselec);//计算成本
                 cost_bitmap_tree_node(pathinfoarray[i]->path, &ocost, &oselec);
                 if (ncost < ocost)
                     pathinfoarray[i] = pathinfo;
             }
             else//没有发现条件一样的,添加到数组中
             {
                 /* not duplicate clauseids, add to array */
                 pathinfoarray[npaths++] = pathinfo;
             }
         }
     
         /* If only one surviving path, we're done */
         if (npaths == 1)//结果只有一条,则返回之
             return pathinfoarray[0]->path;
     
         /* Sort the surviving paths by index access cost */
         qsort(pathinfoarray, npaths, sizeof(PathClauseUsage *),
               path_usage_comparator);//以索引访问成本排序现存路径
     
         /*
          * For each surviving index, consider it as an "AND group leader", and see
          * whether adding on any of the later indexes results in an AND path with
          * cheaper total cost than before.  Then take the cheapest AND group.
          * 对于现存的索引,把它视为"AND group leader",
          * 并查看是否添加了以后的索引后，会得到一个总成本比以前更低的AND路径。
          * 选择成本最低的AND组.
          * 
          */
         for (i = 0; i < npaths; i++)//遍历这些路径
         {
             Cost        costsofar;
             List       *qualsofar;
             Bitmapset  *clauseidsofar;
             ListCell   *lastcell;
     
             pathinfo = pathinfoarray[i];//PathClauseUsage结构体
             paths = list_make1(pathinfo->path);//路径链表
             costsofar = bitmap_scan_cost_est(root, rel, pathinfo->path);//当前的成本
             qualsofar = list_concat(list_copy(pathinfo->quals),
                                     list_copy(pathinfo->preds));
             clauseidsofar = bms_copy(pathinfo->clauseids);
             lastcell = list_head(paths);    /* 用于快速删除,for quick deletions */
     
             for (j = i + 1; j < npaths; j++)//扫描后续的路径
             {
                 Cost        newcost;
     
                 pathinfo = pathinfoarray[j];
                 /* Check for redundancy */
                 if (bms_overlap(pathinfo->clauseids, clauseidsofar))
                     continue;       /* 多余的路径,consider it redundant */
                 if (pathinfo->preds)//部分索引?
                 {
                     bool        redundant = false;
     
                     /* we check each predicate clause separately */
                     //单独检查每一个谓词
                     foreach(l, pathinfo->preds)
                     {
                         Node       *np = (Node *) lfirst(l);
     
                         if (predicate_implied_by(list_make1(np), qualsofar, false))
                         {
                             redundant = true;
                             break;  /* out of inner foreach loop */
                         }
                     }
                     if (redundant)
                         continue;
                 }
                 /* tentatively add new path to paths, so we can estimate cost */
                 //尝试在路径中添加新路径，这样我们就可以估算成本
                 paths = lappend(paths, pathinfo->path);
                 newcost = bitmap_and_cost_est(root, rel, paths);//估算成本
                 if (newcost < costsofar)//新成本更低
                 {
                     /* keep new path in paths, update subsidiary variables */
                     costsofar = newcost;
                     qualsofar = list_concat(qualsofar,
                                             list_copy(pathinfo->quals));//添加此条件
                     qualsofar = list_concat(qualsofar,
                                             list_copy(pathinfo->preds));//添加此谓词
                     clauseidsofar = bms_add_members(clauseidsofar,
                                                     pathinfo->clauseids);//添加此子句ID
                     lastcell = lnext(lastcell);
                 }
                 else
                 {
                     /* reject new path, remove it from paths list */
                     paths = list_delete_cell(paths, lnext(lastcell), lastcell);//去掉新路径
                 }
                 Assert(lnext(lastcell) == NULL);
             }
     
             /* Keep the cheapest AND-group (or singleton) */
             if (i == 0 || costsofar < bestcost)//单条路径或者取得最小的成本
             {
                 bestpaths = paths;
                 bestcost = costsofar;
             }
     
             /* some easy cleanup (we don't try real hard though) */
             list_free(qualsofar);
         }
     
         if (list_length(bestpaths) == 1)
             return (Path *) linitial(bestpaths);    /* 无需AND路径,no need for AND */
         return (Path *) create_bitmap_and_path(root, rel, bestpaths);//生成BitmapAndPath
     }
     
    //-------------------------------------------------------------------------- bitmap_scan_cost_est
     /*
      * Estimate the cost of actually executing a bitmap scan with a single
      * index path (no BitmapAnd, at least not at this level; but it could be
      * a BitmapOr).
      */
     static Cost
     bitmap_scan_cost_est(PlannerInfo *root, RelOptInfo *rel, Path *ipath)
     {
         BitmapHeapPath bpath;
         Relids      required_outer;
     
         /* Identify required outer rels, in case it's a parameterized scan */
         required_outer = get_bitmap_tree_required_outer(ipath);
     
         /* Set up a dummy BitmapHeapPath */
         bpath.path.type = T_BitmapHeapPath;
         bpath.path.pathtype = T_BitmapHeapScan;
         bpath.path.parent = rel;
         bpath.path.pathtarget = rel->reltarget;
         bpath.path.param_info = get_baserel_parampathinfo(root, rel,
                                                           required_outer);
         bpath.path.pathkeys = NIL;
         bpath.bitmapqual = ipath;
     
         /*
          * Check the cost of temporary path without considering parallelism.
          * Parallel bitmap heap path will be considered at later stage.
          */
         bpath.path.parallel_workers = 0;
         cost_bitmap_heap_scan(&bpath.path, root, rel,
                               bpath.path.param_info,
                               ipath,
                               get_loop_count(root, rel->relid, required_outer));//BitmapHeapPath计算成本
     
         return bpath.path.total_cost;
     }
     
    //-------------------------------------------------------------------------- bitmap_and_cost_est
    
     /*
      * Estimate the cost of actually executing a BitmapAnd scan with the given
      * inputs.
      * 给定输入,估算实际执行BitmapAnd扫描的实际成本
      */
     static Cost
     bitmap_and_cost_est(PlannerInfo *root, RelOptInfo *rel, List *paths)
     {
         BitmapAndPath apath;
         BitmapHeapPath bpath;
         Relids      required_outer;
     
         /* Set up a dummy BitmapAndPath */
         apath.path.type = T_BitmapAndPath;
         apath.path.pathtype = T_BitmapAnd;
         apath.path.parent = rel;
         apath.path.pathtarget = rel->reltarget;
         apath.path.param_info = NULL;   /* not used in bitmap trees */
         apath.path.pathkeys = NIL;
         apath.bitmapquals = paths;
         cost_bitmap_and_node(&apath, root);
     
         /* Identify required outer rels, in case it's a parameterized scan */
         required_outer = get_bitmap_tree_required_outer((Path *) &apath);
     
         /* Set up a dummy BitmapHeapPath */
         bpath.path.type = T_BitmapHeapPath;
         bpath.path.pathtype = T_BitmapHeapScan;
         bpath.path.parent = rel;
         bpath.path.pathtarget = rel->reltarget;
         bpath.path.param_info = get_baserel_parampathinfo(root, rel,
                                                           required_outer);
         bpath.path.pathkeys = NIL;
         bpath.bitmapqual = (Path *) &apath;
     
         /*
          * Check the cost of temporary path without considering parallelism.
          * Parallel bitmap heap path will be considered at later stage.
          */
         bpath.path.parallel_workers = 0;
     
         /* Now we can do cost_bitmap_heap_scan */
         cost_bitmap_heap_scan(&bpath.path, root, rel,
                               bpath.path.param_info,
                               (Path *) &apath,
                               get_loop_count(root, rel->relid, required_outer));//BitmapHeapPath计算成本
     
         return bpath.path.total_cost;
     }
     
    
    //-------------------------------------------------------------------------- create_bitmap_and_path
    
     /*
      * create_bitmap_and_path
      *    Creates a path node representing a BitmapAnd.
      */
     BitmapAndPath *
     create_bitmap_and_path(PlannerInfo *root,
                            RelOptInfo *rel,
                            List *bitmapquals)
     {
         BitmapAndPath *pathnode = makeNode(BitmapAndPath);
     
         pathnode->path.pathtype = T_BitmapAnd;
         pathnode->path.parent = rel;
         pathnode->path.pathtarget = rel->reltarget;
         pathnode->path.param_info = NULL;   /* not used in bitmap trees */
     
         /*
          * Currently, a BitmapHeapPath, BitmapAndPath, or BitmapOrPath will be
          * parallel-safe if and only if rel->consider_parallel is set.  So, we can
          * set the flag for this path based only on the relation-level flag,
          * without actually iterating over the list of children.
          */
         pathnode->path.parallel_aware = false;
         pathnode->path.parallel_safe = rel->consider_parallel;
         pathnode->path.parallel_workers = 0;
     
         pathnode->path.pathkeys = NIL;  /* always unordered */
     
         pathnode->bitmapquals = bitmapquals;
     
         /* this sets bitmapselectivity as well as the regular cost fields: */
         cost_bitmap_and_node(pathnode, root);//计算成本
     
         return pathnode;
     }
    
    //----------------------------------------------------- cost_bitmap_and_node
     /*
      * cost_bitmap_and_node
      *      Estimate the cost of a BitmapAnd node
      *      估算BitmapAnd节点成本
      *
      * Note that this considers only the costs of index scanning and bitmap
      * creation, not the eventual heap access.  In that sense the object isn't
      * truly a Path, but it has enough path-like properties (costs in particular)
      * to warrant treating it as one.  We don't bother to set the path rows field,
      * however.
      */
     void
     cost_bitmap_and_node(BitmapAndPath *path, PlannerInfo *root)
     {
         Cost        totalCost;
         Selectivity selec;
         ListCell   *l;
     
         /*
          * We estimate AND selectivity on the assumption that the inputs are
          * independent.  This is probably often wrong, but we don't have the info
          * to do better.
          *
          * The runtime cost of the BitmapAnd itself is estimated at 100x
          * cpu_operator_cost for each tbm_intersect needed.  Probably too small,
          * definitely too simplistic?
          */
         totalCost = 0.0;
         selec = 1.0;
         foreach(l, path->bitmapquals)
         {
             Path       *subpath = (Path *) lfirst(l);
             Cost        subCost;
             Selectivity subselec;
     
             cost_bitmap_tree_node(subpath, &subCost, &subselec);
     
             selec *= subselec;
     
             totalCost += subCost;
             if (l != list_head(path->bitmapquals))
                 totalCost += 100.0 * cpu_operator_cost;
         }
         path->bitmapselectivity = selec;
         path->path.rows = 0;        /* per above, not used */
         path->path.startup_cost = totalCost;
         path->path.total_cost = totalCost;
     }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    select t1.* 
    from t_dwxx t1 
    where (dwbh > '10000' and dwbh < '15000') AND (dwdz between 'DWDZ10000' and 'DWDZ15000');
    

启动gdb跟踪

    
    
    (gdb) b choose_bitmap_and
    Breakpoint 1 at 0x74e8c2: file indxpath.c, line 1372.
    (gdb) c
    Continuing.
    
    Breakpoint 1, choose_bitmap_and (root=0x1666638, rel=0x1666a48, paths=0x166fdf0) at indxpath.c:1372
    1372    int     npaths = list_length(paths);
    

输入参数

    
    
    (gdb) p *paths
    $1 = {type = T_List, length = 2, head = 0x166fe20, tail = 0x16706b8}
    (gdb) p *(Node *)paths->head->data.ptr_value
    $2 = {type = T_IndexPath}
    (gdb) p *(Node *)paths->head->next->data.ptr_value
    $3 = {type = T_IndexPath}
    (gdb) set $p1=(IndexPath *)paths->head->data.ptr_value
    (gdb) set $p2=(IndexPath *)paths->head->next->data.ptr_value
    (gdb) p *$p1
    $4 = {path = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x1666a48, pathtarget = 0x166d988, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 33, startup_cost = 0.28500000000000003, 
        total_cost = 116.20657683302848, pathkeys = 0x0}, indexinfo = 0x166e420, indexclauses = 0x166f528, 
      indexquals = 0x166f730, indexqualcols = 0x166f780, indexorderbys = 0x0, indexorderbycols = 0x0, 
      indexscandir = ForwardScanDirection, indextotalcost = 18.205000000000002, indexselectivity = 0.059246954595791879}
    (gdb) p *$p2
    $5 = {path = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x1666a48, pathtarget = 0x166d988, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 33, startup_cost = 0.28500000000000003, 
        total_cost = 111.33157683302848, pathkeys = 0x0}, indexinfo = 0x1666c58, indexclauses = 0x166fed0, 
      indexquals = 0x166ffc8, indexqualcols = 0x1670018, indexorderbys = 0x0, indexorderbycols = 0x0, 
      indexscandir = ForwardScanDirection, indextotalcost = 13.855, indexselectivity = 0.055688888888888899}
    

paths中的第1个元素对应(dwbh > '10000' and dwbh < '15000') ,第2个元素对应(dwdz between
'DWDZ10000' and 'DWDZ15000')

    
    
    (gdb) set $ri1=(RestrictInfo *)$p1->indexclauses->head->data.ptr_value
    (gdb) set $tmp=(RelabelType *)((OpExpr *)$ri1->clause)->args->head->data.ptr_value
    (gdb) p *(Var *)$tmp->arg
    $17 = {xpr = {type = T_Var}, varno = 1, varattno = 3, vartype = 1043, vartypmod = 104, varcollid = 100, varlevelsup = 0, 
      varnoold = 1, varoattno = 3, location = 76}
    (gdb) p *(Node *)((OpExpr *)$ri1->clause)->args->head->next->data.ptr_value
    $18 = {type = T_Const}
    (gdb) p *(Const *)((OpExpr *)$ri1->clause)->args->head->next->data.ptr_value
    $19 = {xpr = {type = T_Const}, consttype = 25, consttypmod = -1, constcollid = 100, constlen = -1, constvalue = 23636608, 
      constisnull = false, constbyval = false, location = 89}
    

开始遍历paths,提取子句条件并检测是否使用完全相同子句集的所有路径,只保留这些路径中成本最低的,这些路径被放入一个数组中进行qsort.

    
    
    ...
    (gdb) 
    1444    npaths = 0;
    (gdb) 
    1445    foreach(l, paths)
    (gdb) 
    
    

收集信息到PathClauseUsage数组中

    
    
    ...
    (gdb) n
    1471        pathinfoarray[npaths++] = pathinfo;
    (gdb) 
    1445    foreach(l, paths)
    (gdb) 
    1476    if (npaths == 1)
    (gdb) p npaths
    $26 = 2
    (gdb) 
    

按成本排序

    
    
    (gdb) n
    1480    qsort(pathinfoarray, npaths, sizeof(PathClauseUsage *),
    

遍历路径,找到成本最低的AND group

    
    
    1488    for (i = 0; i < npaths; i++)
    (gdb) n
    1495      pathinfo = pathinfoarray[i];
    (gdb) 
    1496      paths = list_make1(pathinfo->path);
    (gdb) 
    1497      costsofar = bitmap_scan_cost_est(root, rel, pathinfo->path);
    (gdb) 
    1499                  list_copy(pathinfo->preds));
    

获取当前的成本,设置当前的条件子句

    
    
    (gdb) p costsofar
    $27 = 89.003250000000008
    (gdb) n
    1498      qualsofar = list_concat(list_copy(pathinfo->quals),
    

执行AND操作(路径叠加),成本更低,调整当前成本和相关变量

    
    
    (gdb) n
    1531        newcost = bitmap_and_cost_est(root, rel, paths);
    (gdb) 
    1532        if (newcost < costsofar)
    (gdb) p newcost
    $30 = 88.375456720095343
    (gdb) n
    1535          costsofar = newcost;
    (gdb) n
    1537                      list_copy(pathinfo->quals));
    (gdb) 
    1536          qualsofar = list_concat(qualsofar,
    (gdb) 
    1539                      list_copy(pathinfo->preds));
    

处理下一个AND条件,单个AND条件比上一个条件成本高,保留原来的

    
    
    1488    for (i = 0; i < npaths; i++)
    (gdb) 
    1495      pathinfo = pathinfoarray[i];
    (gdb) 
    1496      paths = list_make1(pathinfo->path);
    (gdb) 
    1497      costsofar = bitmap_scan_cost_est(root, rel, pathinfo->path);
    (gdb) 
    1499                  list_copy(pathinfo->preds));
    (gdb) p costsofar
    $34 = 94.053250000000006
    (gdb) n
    1498      qualsofar = list_concat(list_copy(pathinfo->quals),
    (gdb) 
    1500      clauseidsofar = bms_copy(pathinfo->clauseids);
    (gdb) 
    1501      lastcell = list_head(paths);  /* for quick deletions */
    (gdb) 
    1503      for (j = i + 1; j < npaths; j++)
    (gdb) 
    1553      if (i == 0 || costsofar < bestcost)
    (gdb) p i
    $35 = 1
    (gdb) p costsofar
    $36 = 94.053250000000006
    (gdb) p bestcost
    $37 = 88.375456720095343
    (gdb) 
    

构建BitmapAndPath,返回

    
    
    (gdb) n
    1563    if (list_length(bestpaths) == 1)
    (gdb) 
    1565    return (Path *) create_bitmap_and_path(root, rel, bestpaths);
    (gdb) 
    1566  }
    

DONE!

    
    
    (gdb) n
    create_index_paths (root=0x1666638, rel=0x1666a48) at indxpath.c:337
    337     bpath = create_bitmap_heap_path(root, rel, bitmapqual,
    

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

