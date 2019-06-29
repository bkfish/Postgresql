本小节继续介绍查询物理优化中的create_index_paths->generate_bitmap_or_paths函数,该函数从条件子句链表中寻找OR子句,如找到并且可以处理则生成BitmapOrPath。  
关于Bitmap Scan的相关知识,请参照[PostgreSQL DBA(6) - SeqScan vs IndexScan vs
BitmapHeapScan](https://www.jianshu.com/p/4f819a43882d)这篇文章。

下面是BitmapOrPath访问路径样例:

    
    
    testdb=# explain verbose select t1.* from t_dwxx t1 where (dwbh > '10000' and dwbh < '30000') OR (dwdz between 'DWDZ10000' and 'DWDZ20000');
    QUERY PLAN                              
                                                           
    ---------------------------------------------------------------------------------------------     Bitmap Heap Scan on public.t_dwxx t1  (cost=84.38..216.82 rows=3156 width=20)
       Output: dwmc, dwbh, dwdz
       Recheck Cond: ((((t1.dwbh)::text > '10000'::text) AND ((t1.dwbh)::text < '30000'::text)) OR (((t1.dwdz)::text >= 'DWDZ1000
    0'::text) AND ((t1.dwdz)::text <= 'DWDZ20000'::text)))
       ->  BitmapOr  (cost=84.38..84.38 rows=3422 width=0)  -->BitmapOr
             ->  Bitmap Index Scan on t_dwxx_pkey  (cost=0.00..50.52 rows=2223 width=0)
                   Index Cond: (((t1.dwbh)::text > '10000'::text) AND ((t1.dwbh)::text < '30000'::text))
             ->  Bitmap Index Scan on idx_dwxx_dwdz  (cost=0.00..32.28 rows=1200 width=0)
                   Index Cond: (((t1.dwdz)::text >= 'DWDZ10000'::text) AND ((t1.dwdz)::text <= 'DWDZ20000'::text))
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
    

### 二、源码解读

**generate_bitmap_or_paths函数**  
create_index_paths->generate_bitmap_or_paths函数从条件子句链表中寻找OR子句,如找到并且可以处理则生成BitmapOrPath.函数返回生成的链表BitmapOrPaths.

    
    
     /*
      * generate_bitmap_or_paths
      *      Look through the list of clauses to find OR clauses, and generate
      *      a BitmapOrPath for each one we can handle that way.  Return a list
      *      of the generated BitmapOrPaths.
      *   从条件子句链表中寻找OR子句,如找到并且可以处理则生成BitmapOrPath.
      *     函数返回生成的链表BitmapOrPaths 
      *
      * other_clauses is a list of additional clauses that can be assumed true
      * for the purpose of generating indexquals, but are not to be searched for
      * ORs.  (See build_paths_for_OR() for motivation.)
      * other_clauses是一个附加子句链表，
      * 为了生成索引条件，可以假定为true，但不能用于搜索OR子句。
      */
     static List *
     generate_bitmap_or_paths(PlannerInfo *root, RelOptInfo *rel,
                              List *clauses, List *other_clauses)
     {
         List       *result = NIL;
         List       *all_clauses;
         ListCell   *lc;
     
         /*
          * We can use both the current and other clauses as context for
          * build_paths_for_OR; no need to remove ORs from the lists.
          * 使用当前和其他子句作为build_paths_for_OR函数的输入参数
          * 从而不需要从列表中删除OR子句。
          */
         all_clauses = list_concat(list_copy(clauses), other_clauses);//合并到链表中
     
         foreach(lc, clauses)//遍历子句链表
         {
             RestrictInfo *rinfo = lfirst_node(RestrictInfo, lc);//约束条件
             List       *pathlist;//路径链表
             Path       *bitmapqual;//
             ListCell   *j;
     
             /* Ignore RestrictInfos that aren't ORs */
             if (!restriction_is_or_clause(rinfo))//不是OR子句,处理下一个子句
                 continue;
     
             /*
              * We must be able to match at least one index to each of the arms of
              * the OR, else we can't use it.
              * 必须能够将至少一个索引匹配到OR的某个分支，否则无法使用索引。
              */
             pathlist = NIL;
             foreach(j, ((BoolExpr *) rinfo->orclause)->args)//遍历OR子句参数
             {
                 Node       *orarg = (Node *) lfirst(j);//参数节点
                 List       *indlist;
     
                 /* OR arguments should be ANDs or sub-RestrictInfos */
                 //OR子句的参数必须是AND子句或者是子约束条件
                 if (and_clause(orarg))//如为AND子句
                 {
                     List       *andargs = ((BoolExpr *) orarg)->args;//获取AND子句的参数
     
                     indlist = build_paths_for_OR(root, rel,
                                                  andargs,
                                                  all_clauses);//构建路径
     
                     /* Recurse in case there are sub-ORs */
                     //递归调用generate_bitmap_or_paths,并添加到访问路径链表中
                     indlist = list_concat(indlist,
                                           generate_bitmap_or_paths(root, rel,
                                                                    andargs,
                                                                    all_clauses));
                 }
                 else
                 {
                     RestrictInfo *rinfo = castNode(RestrictInfo, orarg);//不是AND,则为约束条件
                     List       *orargs;
     
                     Assert(!restriction_is_or_clause(rinfo));
                     orargs = list_make1(rinfo);
     
                     indlist = build_paths_for_OR(root, rel,
                                                  orargs,
                                                  all_clauses);//构建访问路径
                 }
     
                 /*
                  * If nothing matched this arm, we can't do anything with this OR
                  * clause.
                  */
                 if (indlist == NIL)
                 {
                     pathlist = NIL;
                     break;
                 }
     
                 /*
                  * OK, pick the most promising AND combination, and add it to
                  * pathlist.
                  * 选择最有希望的组合，并将其添加到路径列表中。
                  */
                 bitmapqual = choose_bitmap_and(root, rel, indlist);
                 pathlist = lappend(pathlist, bitmapqual);
             }
     
             /*
              * If we have a match for every arm, then turn them into a
              * BitmapOrPath, and add to result list.
              * 如果左右两边都匹配，那么将它们转换为BitmapOrPath，并添加到结果列表中。
              */
             if (pathlist != NIL)
             {
           //创建BitmapOrPath
                 bitmapqual = (Path *) create_bitmap_or_path(root, rel, pathlist);
                 result = lappend(result, bitmapqual);
             }
         }
     
         return result;
     }
     
    //------------------------------------------------------ build_paths_for_OR
     /*
      * build_paths_for_OR
      *    Given a list of restriction clauses from one arm of an OR clause,
      *    construct all matching IndexPaths for the relation.
      *    给定OR子句的约束条件子句,构建该Relation所有匹配的索引访问路径.
      *
      * Here we must scan all indexes of the relation, since a bitmap OR tree
      * can use multiple indexes.
      * BitmapOr可能会使用多个索引,因此需要访问该Relation的所有索引.
      *
      * The caller actually supplies two lists of restriction clauses: some
      * "current" ones and some "other" ones.  Both lists can be used freely
      * to match keys of the index, but an index must use at least one of the
      * "current" clauses to be considered usable.  The motivation for this is
      * examples like
      *      WHERE (x = 42) AND (... OR (y = 52 AND z = 77) OR ....)
      * While we are considering the y/z subclause of the OR, we can use "x = 42"
      * as one of the available index conditions; but we shouldn't match the
      * subclause to any index on x alone, because such a Path would already have
      * been generated at the upper level.  So we could use an index on x,y,z
      * or an index on x,y for the OR subclause, but not an index on just x.
      * When dealing with a partial index, a match of the index predicate to
      * one of the "current" clauses also makes the index usable.
      * 函数调用方提供了2个约束条件子句链表,一个是"current",另外一个是"other".
      * 这两个链表都可以用于资源匹配索引键,但是索引必须使用至少一个存在于"current"中的子句. 
      * 举个例子,有下面的条件语句:
      *     WHERE (x = 42) AND (... OR (y = 52 AND z = 77) OR ....)
      * 在考察OR中的x/z子句时,可以使用"x = 42"作为可用的索引条件,但
      * 但不应该将子句单独与x上的任何索引进行匹配，因为这样的访问路径已经在上层生成。
      * 因此可以在OR子句上使用x,y,z上索引或x,y上的索引,但不能是x上的索引.
      * 在处理部分索引时,与索引谓词匹配的"current"子句同样可以使用此索引.
      *
      * 'rel' is the relation for which we want to generate index paths
      * 'clauses' is the current list of clauses (RestrictInfo nodes)
      * 'other_clauses' is the list of additional upper-level clauses
      * 输入参数:
      * rel-需要生成访问路径的Relation 
      * clauses-"current"子句,节点类型为RestrictInfo
      * other_clauses-"other"子句,上层子句已处理
      */
     static List *
     build_paths_for_OR(PlannerInfo *root, RelOptInfo *rel,
                        List *clauses, List *other_clauses)
     {
         List       *result = NIL;//返回结果
         List       *all_clauses = NIL;  /* not computed till needed */
         ListCell   *lc;//临时变量
     
         foreach(lc, rel->indexlist)//遍历RelOptInfo上的Index
         {
             IndexOptInfo *index = (IndexOptInfo *) lfirst(lc);//IndexOptInfo
             IndexClauseSet clauseset;//
             List       *indexpaths;
             bool        useful_predicate;
     
             /* Ignore index if it doesn't support bitmap scans */
             if (!index->amhasgetbitmap)//索引不支持BitmapIndexScan
                 continue;
     
             /*
              * Ignore partial indexes that do not match the query.  If a partial
              * index is marked predOK then we know it's OK.  Otherwise, we have to
              * test whether the added clauses are sufficient to imply the
              * predicate. If so, we can use the index in the current context.
              * 忽略不匹配查询的部分索引。如predOK标记为T,则可考虑使用此索引.
              * 否则，必须测试添加的子句是否足以包含索引谓词.
              * 在这种情况下才可以在当前上下文中使用索引。
              *
              * We set useful_predicate to true iff the predicate was proven using
              * the current set of clauses.  This is needed to prevent matching a
              * predOK index to an arm of an OR, which would be a legal but
              * pointlessly inefficient plan.  (A better plan will be generated by
              * just scanning the predOK index alone, no OR.)
              * 如验证通过,则将useful_predicate设置为T。
          * 这是为了避免predOK索引与OR的某个分支相匹配，这是一个合法但无意义的低效计划。
          * (只需要扫描部分索引就可以产生一个更好的计划，但不是OR子句)
              */
             useful_predicate = false;
             if (index->indpred != NIL)
             {
                 if (index->predOK)
                 {
                     /* Usable, but don't set useful_predicate */
                 }
                 else
                 {
                     /* Form all_clauses if not done already */
                     if (all_clauses == NIL)
                         all_clauses = list_concat(list_copy(clauses),
                                                   other_clauses);
     
                     if (!predicate_implied_by(index->indpred, all_clauses, false))
                         continue;   /* can't use it at all */
     
                     if (!predicate_implied_by(index->indpred, other_clauses, false))
                         useful_predicate = true;
                 }
             }
     
             /*
              * Identify the restriction clauses that can match the index.
              * 标记与索引匹配的约束条件子句
              */
             MemSet(&clauseset, 0, sizeof(clauseset));
             match_clauses_to_index(index, clauses, &clauseset);
     
             /*
              * If no matches so far, and the index predicate isn't useful, we
              * don't want it.
              */
             if (!clauseset.nonempty && !useful_predicate)//没有合适的,继续下一个索引
                 continue;
     
             /*
              * Add "other" restriction clauses to the clauseset.
              */
             match_clauses_to_index(index, other_clauses, &clauseset);//添加到clauseset中
     
             /*
              * Construct paths if possible.
              */
             indexpaths = build_index_paths(root, rel,
                                            index, &clauseset,
                                            useful_predicate,
                                            ST_BITMAPSCAN,
                                            NULL,
                                            NULL);//构建索引访问路径
             result = list_concat(result, indexpaths);
         }
     
         return result;
     }
     
    
    //------------------------------------------------------ create_bitmap_or_path
    
     /*
      * create_bitmap_or_path
      *    Creates a path node representing a BitmapOr.
      *    创建BitmapOr路径节点
      */
     BitmapOrPath *
     create_bitmap_or_path(PlannerInfo *root,
                           RelOptInfo *rel,
                           List *bitmapquals)
     {
         BitmapOrPath *pathnode = makeNode(BitmapOrPath);
     
         pathnode->path.pathtype = T_BitmapOr;
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
         cost_bitmap_or_node(pathnode, root);//计算成本
     
         return pathnode;
     }
    
    //------------------------------------ create_bitmap_or_path
    
     /*
      * cost_bitmap_or_node
      *      Estimate the cost of a BitmapOr node
      *      估算BitmapOr成本
      *
      * See comments for cost_bitmap_and_node.
      */
     void
     cost_bitmap_or_node(BitmapOrPath *path, PlannerInfo *root)
     {
         Cost        totalCost;
         Selectivity selec;
         ListCell   *l;
     
         /*
          * We estimate OR selectivity on the assumption that the inputs are
          * non-overlapping, since that's often the case in "x IN (list)" type
          * situations.  Of course, we clamp to 1.0 at the end.
          * 我们估算或计算选择率的前提是输入不重叠，因为存在“x in (list)”这样的情况。
          * 当然，我们在最后调整为1.0。
        *
          * The runtime cost of the BitmapOr itself is estimated at 100x
          * cpu_operator_cost for each tbm_union needed.  Probably too small,
          * definitely too simplistic?  We are aware that the tbm_unions are
          * optimized out when the inputs are BitmapIndexScans.
          * 对于所需的每个tbm_union操作, 
          * BitmapOr本身的运行时成本估计为100 x cpu_operator_cost。
          * 这个估值是否太小，太简单了?其实，当输入是位图索引扫描时，tbm_unions已被优化。
          */
         totalCost = 0.0;//成本
         selec = 0.0;//选择率
         foreach(l, path->bitmapquals)//遍历条件
         {
             Path       *subpath = (Path *) lfirst(l);//路径
             Cost        subCost;//成本
             Selectivity subselec;
     
             cost_bitmap_tree_node(subpath, &subCost, &subselec);//遍历路径获取成本&选择率
     
             selec += subselec;//
     
             totalCost += subCost;
             if (l != list_head(path->bitmapquals) &&
                 !IsA(subpath, IndexPath))
                 totalCost += 100.0 * cpu_operator_cost;//非单个条件而且不是索引访问路径,则添加运行期成本
         }
         path->bitmapselectivity = Min(selec, 1.0);//选择率
         path->path.rows = 0;        /* per above, not used */
         path->path.startup_cost = totalCost;
         path->path.total_cost = totalCost;
     }
     
    

### 三、跟踪分析

测试脚本如下

    
    
    select t1.* 
    from t_dwxx t1 
    where (dwbh > '10000' and dwbh < '30000') 
          OR (dwdz between 'DWDZ10000' and 'DWDZ20000');
    

启动gdb跟踪

    
    
    (gdb) b generate_bitmap_or_paths
    Breakpoint 1 at 0x74e6c1: file indxpath.c, line 1266.
    (gdb) c
    Continuing.
    
    Breakpoint 1, generate_bitmap_or_paths (root=0x2aa6248, rel=0x2aa6658, clauses=0x2aaf138, other_clauses=0x0)
        at indxpath.c:1266
    1266    List     *result = NIL;
    

查看输入参数,clauses是链表,只有一个元素(BoolExpr类型,即OR子句);other_clauses为NULL

    
    
    (gdb) p *clauses
    $1 = {type = T_List, length = 1, head = 0x2aaf118, tail = 0x2aaf118}
    (gdb) p *(Node *)clauses->head->data.ptr_value
    $2 = {type = T_RestrictInfo}
    (gdb) p *(RestrictInfo *)clauses->head->data.ptr_value
    $3 = {type = T_RestrictInfo, clause = 0x2aad818, is_pushed_down = true, outerjoin_delayed = false, can_join = false, 
      pseudoconstant = false, leakproof = false, security_level = 0, clause_relids = 0x2aaf100, required_relids = 0x2aae938, 
      outer_relids = 0x0, nullable_relids = 0x0, left_relids = 0x0, right_relids = 0x0, orclause = 0x2aaefc0, parent_ec = 0x0, 
      eval_cost = {startup = 0, per_tuple = 0.01}, norm_selec = 0.31556115090433856, outer_selec = -1, mergeopfamilies = 0x0, 
      left_ec = 0x0, right_ec = 0x0, left_em = 0x0, right_em = 0x0, scansel_cache = 0x0, outer_is_left = false, 
      hashjoinoperator = 0, left_bucketsize = -1, right_bucketsize = -1, left_mcvfreq = -1, right_mcvfreq = -1}
    (gdb) p *((RestrictInfo *)clauses->head->data.ptr_value)->clause
    $4 = {type = T_BoolExpr}
    (gdb) set $clause=((RestrictInfo *)clauses->head->data.ptr_value)->clause
    (gdb) p *(BoolExpr *)$clause
    $6 = {xpr = {type = T_BoolExpr}, boolop = OR_EXPR, args = 0x2aad758, location = -1}
    

遍历clausees子句,rinfo->clause即BoolExpr(OR子句)

    
    
    ...
    1276    foreach(lc, clauses)
    (gdb) 
    1278      RestrictInfo *rinfo = lfirst_node(RestrictInfo, lc);
    (gdb) 
    1284      if (!restriction_is_or_clause(rinfo))
    

遍历OR子句的参数

    
    
    (gdb) 
    1292      foreach(j, ((BoolExpr *) rinfo->orclause)->args)
    

参数的第一个元素,BoolExpr,boolop操作符为AND_EXPR

    
    
    (gdb) n
    1294        Node     *orarg = (Node *) lfirst(j);
    (gdb) 
    1298        if (and_clause(orarg))
    (gdb) p *orarg
    $10 = {type = T_BoolExpr}
    (gdb) p *(BoolExpr *)orarg
    $11 = {xpr = {type = T_BoolExpr}, boolop = AND_EXPR, args = 0x2aaea90, location = -1}
    

AND子句的参数

    
    
    (gdb) n
    1300          List     *andargs = ((BoolExpr *) orarg)->args;
    (gdb) 
    1302          indlist = build_paths_for_OR(root, rel,
    (gdb) p *andargs
    $12 = {type = T_List, length = 2, head = 0x2aada78, tail = 0x2aada98}
    (gdb) p *(Node *)andargs->head->data.ptr_value
    $13 = {type = T_RestrictInfo}
    (gdb) p *(RestrictInfo *)andargs->head->data.ptr_value
    $14 = {type = T_RestrictInfo, clause = 0x2aace08, is_pushed_down = true, outerjoin_delayed = false, can_join = false, 
      pseudoconstant = false, leakproof = false, security_level = 0, clause_relids = 0x2aaea78, required_relids = 0x2aaea78, 
      outer_relids = 0x0, nullable_relids = 0x0, left_relids = 0x2aaea60, right_relids = 0x0, orclause = 0x0, parent_ec = 0x0, 
      eval_cost = {startup = 0, per_tuple = 0.0025000000000000001}, norm_selec = 0.99990000000000001, outer_selec = -1, 
      mergeopfamilies = 0x0, left_ec = 0x0, right_ec = 0x0, left_em = 0x0, right_em = 0x0, scansel_cache = 0x0, 
      outer_is_left = false, hashjoinoperator = 0, left_bucketsize = -1, right_bucketsize = -1, left_mcvfreq = -1, 
      right_mcvfreq = -1}
    (gdb) p *((RestrictInfo *)andargs->head->data.ptr_value)->clause
    $15 = {type = T_OpExpr}
    (gdb) set $tmp=((RestrictInfo *)andargs->head->data.ptr_value)->clause
    (gdb) p *(OpExpr *)$tmp
    $16 = {xpr = {type = T_OpExpr}, opno = 666, opfuncid = 742, opresulttype = 16, opretset = false, opcollid = 0, 
      inputcollid = 100, args = 0x2aacd68, location = 39}
    (gdb) set $tmp2=((RestrictInfo *)andargs->head->next->data.ptr_value)->clause
    (gdb) p *(OpExpr *)$tmp2
    $17 = {xpr = {type = T_OpExpr}, opno = 664, opfuncid = 740, opresulttype = 16, opretset = false, opcollid = 0, 
      inputcollid = 100, args = 0x2aacc78, location = 58}
    (gdb) 
    

进入build_paths_for_OR函数

    
    
    (gdb) step
    build_paths_for_OR (root=0x2aa6248, rel=0x2aa6658, clauses=0x2aaea90, other_clauses=0x2aaf598) at indxpath.c:1170
    1170    List     *result = NIL;
    

遍历索引,第一个索引是idx_dwxx_dwdz

    
    
    1174    foreach(lc, rel->indexlist)
    (gdb) 
    1176      IndexOptInfo *index = (IndexOptInfo *) lfirst(lc);
    (gdb) 
    1182      if (!index->amhasgetbitmap)
    (gdb) p *index
    $18 = {type = T_IndexOptInfo, indexoid = 16753, reltablespace = 0, rel = 0x2aa6658, pages = 40, tuples = 10000, 
      tree_height = 1, ncolumns = 1, nkeycolumns = 1, indexkeys = 0x2aae590, indexcollations = 0x2aae5a8, opfamily = 0x2aae5c0, 
      opcintype = 0x2aae5d8, sortopfamily = 0x2aae5c0, reverse_sort = 0x2aae608, nulls_first = 0x2aae620, 
      canreturn = 0x2aae5f0, relam = 403, indexprs = 0x0, indpred = 0x0, indextlist = 0x2aae6f8, indrestrictinfo = 0x2aaf138, 
      predOK = false, unique = false, immediate = true, hypothetical = false, amcanorderbyop = false, amoptionalkey = true, 
      amsearcharray = true, amsearchnulls = true, amhasgettuple = true, amhasgetbitmap = true, amcanparallel = true, 
      amcostestimate = 0x94f0ad <btcostestimate>}
    --    testdb=# select relname from pg_class where oid=16753;
        relname    
    ---------------     idx_dwxx_dwdz
    (1 row)
    --  
    

与约束条件不匹配((dwbh > '10000' and dwbh < '30000')),继续下一个索引

    
    
    1229      if (!clauseset.nonempty && !useful_predicate)
    (gdb) p clauseset
    $20 = {nonempty = false, indexclauses = {0x0 <repeats 32 times>}}
    (gdb) n
    1230        continue;
    

下一个索引是idx_dwxx_predicate_dwmc/idx_dwxx_expr,同样不匹配,继续寻找索引,直至索引t_dwxx_pkey

    
    
    (gdb) p *index
    $23 = {type = T_IndexOptInfo, indexoid = 16738,...
    1223      match_clauses_to_index(index, clauses, &clauseset);
    (gdb) 
    1229      if (!clauseset.nonempty && !useful_predicate)
    (gdb) p clauseset
    $24 = {nonempty = true, indexclauses = {0x2aaf638, 0x0 <repeats 31 times>}}
    

构建索引访问路径

    
    
    (gdb) 
    1246      result = list_concat(result, indexpaths);
    (gdb) p *indexpaths
    $25 = {type = T_List, length = 1, head = 0x2aafb48, tail = 0x2aafb48}
    (gdb) p *(Node *)indexpaths->head->data.ptr_value
    $26 = {type = T_IndexPath}
    (gdb) p *(IndexPath *)indexpaths->head->data.ptr_value
    $27 = {path = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2aa6658, pathtarget = 0x2aad8d8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 3156, startup_cost = 0.28500000000000003, 
        total_cost = 191.46871600907946, pathkeys = 0x0}, indexinfo = 0x2aa6868, indexclauses = 0x2aaf6a8, 
      indexquals = 0x2aaf898, indexqualcols = 0x2aaf8e8, indexorderbys = 0x0, indexorderbycols = 0x0, 
      indexscandir = ForwardScanDirection, indextotalcost = 50.515000000000001, indexselectivity = 0.22227191011235958}
    

回到generate_bitmap_or_paths函数

    
    
    1250  }
    (gdb) 
    generate_bitmap_or_paths (root=0x2aa6248, rel=0x2aa6658, clauses=0x2aaf138, other_clauses=0x0) at indxpath.c:1307
    1307          indlist = list_concat(indlist,
    

递归进入generate_bitmap_or_paths

    
    
    (gdb) n
    
    Breakpoint 1, generate_bitmap_or_paths (root=0x2aa6248, rel=0x2aa6658, clauses=0x2aaea90, other_clauses=0x2aaf598)
        at indxpath.c:1266
    1266    List     *result = NIL;
    #直接结束
    (gdb) finish
    Run till exit from #0  generate_bitmap_or_paths (root=0x2aa6248, rel=0x2aa6658, clauses=0x2aaea90, other_clauses=0x2aaf598)
        at indxpath.c:1266
    0x000000000074e7a0 in generate_bitmap_or_paths (root=0x2aa6248, rel=0x2aa6658, clauses=0x2aaf138, other_clauses=0x0)
        at indxpath.c:1307
    1307          indlist = list_concat(indlist,
    Value returned is $28 = (List *) 0x0
    
    

完成第一轮循环

    
    
    (gdb) n
    1329        if (indlist == NIL)
    (gdb) n
    1339        bitmapqual = choose_bitmap_and(root, rel, indlist);
    (gdb) 
    1340        pathlist = lappend(pathlist, bitmapqual);
    (gdb) p *bitmapqual
    $29 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2aa6658, pathtarget = 0x2aad8d8, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 3156, startup_cost = 0.28500000000000003, 
      total_cost = 191.46871600907946, pathkeys = 0x0}
    

这是第二个AND子句

    
    
    1292      foreach(j, ((BoolExpr *) rinfo->orclause)->args)
    (gdb) 
    1294        Node     *orarg = (Node *) lfirst(j);
    (gdb) 
    1298        if (and_clause(orarg))
    (gdb) 
    1300          List     *andargs = ((BoolExpr *) orarg)->args;
    

完成第二轮循环

    
    
    (gdb) 
    1339        bitmapqual = choose_bitmap_and(root, rel, indlist);
    (gdb) 
    1340        pathlist = lappend(pathlist, bitmapqual);
    (gdb) p bitmapqual
    $33 = (Path *) 0x2aafd78
    (gdb) p *bitmapqual
    $34 = {type = T_IndexPath, pathtype = T_IndexScan, parent = 0x2aa6658, pathtarget = 0x2aad8d8, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 3156, startup_cost = 0.28500000000000003, 
      total_cost = 148.08735471522883, pathkeys = 0x0}
    

结束循环,构建BitmapOrPath

    
    
    1347      if (pathlist != NIL)
    (gdb) 
    1349        bitmapqual = (Path *) create_bitmap_or_path(root, rel, pathlist);
    

进入create_bitmap_or_path,调用函数cost_bitmap_or_node计算成本

    
    
    (gdb) step
    create_bitmap_or_path (root=0x2aa6248, rel=0x2aa6658, bitmapquals=0x2aafbf8) at pathnode.c:1156
    1156    BitmapOrPath *pathnode = makeNode(BitmapOrPath);
    ...
    1178    cost_bitmap_or_node(pathnode, root);
    (gdb) step
    cost_bitmap_or_node (path=0x2ab0278, root=0x2aa6248) at costsize.c:1149
    ...
    

计算结果,与执行计划中的信息相匹配"BitmapOr (cost=84.38..84.38 rows=3422 width=0)"

    
    
    (gdb) p *path
    $37 = {path = {type = T_BitmapOrPath, pathtype = T_BitmapOr, parent = 0x2aa6658, pathtarget = 0x2aad8d8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 0, startup_cost = 84.378, 
        total_cost = 84.378, pathkeys = 0x0}, bitmapquals = 0x2aafbf8, bitmapselectivity = 0.34222288270157986}
    

回到generate_bitmap_or_paths

    
    
    (gdb) n
    create_bitmap_or_path (root=0x2aa6248, rel=0x2aa6658, bitmapquals=0x2aafbf8) at pathnode.c:1180
    1180    return pathnode;
    (gdb) 
    1181  }
    (gdb) 
    generate_bitmap_or_paths (root=0x2aa6248, rel=0x2aa6658, clauses=0x2aaf138, other_clauses=0x0) at indxpath.c:1350
    1350        result = lappend(result, bitmapqual);
    

完成,DONE!

    
    
    (gdb) n
    1276    foreach(lc, clauses)
    (gdb) 
    1354    return result;
    (gdb) 
    1355  }
    

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

