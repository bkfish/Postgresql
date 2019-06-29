本节继续介绍make_one_rel函数中的set_base_rel_pathlists->create_tidscan_paths函数，该函数创建相应的TID扫描路径。

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

set_base_rel_pathlists->create_tidscan_paths函数创建相应的TID扫描路径。

    
    
    /*
     * create_tidscan_paths
     *    Create paths corresponding to direct TID scans of the given rel.
     *    创建相应的TID扫描路径
     *
     *    Candidate paths are added to the rel's pathlist (using add_path).
     *    候选路径会添加到关系的pathlist链表中
     */
    void
    create_tidscan_paths(PlannerInfo *root, RelOptInfo *rel)
    {
        Relids      required_outer;
        List       *tidquals;
    
        /*
         * We don't support pushing join clauses into the quals of a tidscan, but
         * it could still have required parameterization due to LATERAL refs in
         * its tlist.
         */
        required_outer = rel->lateral_relids;//需依赖的外部relids
    
        tidquals = TidQualFromBaseRestrictinfo(rel);//tid条件子句
    
        if (tidquals)
            add_path(rel, (Path *) create_tidscan_path(root, rel, tidquals,
                                                       required_outer));//添加tid路径(如有)
    }
    
    //-------------------------------------------------------------------------- TidQualFromBaseRestrictinfo
    /*
     *  Extract a set of CTID conditions from the rel's baserestrictinfo list
     *  在关系的约束条件链表中抽取CTID条件集合
     */
    static List *
    TidQualFromBaseRestrictinfo(RelOptInfo *rel)
    {
        List       *rlst = NIL;
        ListCell   *l;
    
        foreach(l, rel->baserestrictinfo)//循环遍历约束条件
        {
            RestrictInfo *rinfo = (RestrictInfo *) lfirst(l);//约束条件
    
            /*
             * If clause must wait till after some lower-security-level
             * restriction clause, reject it.
             */
            if (!restriction_is_securely_promotable(rinfo, rel))
                continue;
    
            rlst = TidQualFromExpr((Node *) rinfo->clause, rel->relid);//获取结果链表
            if (rlst)
                break;//如有,则退出
        }
        return rlst;
    }
    
    //------------------------------------------------------ TidQualFromExpr
    
     /*
      *  Extract a set of CTID conditions from the given qual expression
      *  给定条件表达式,获取CTID条件集合
      *
      *  Returns a List of CTID qual expressions (with implicit OR semantics
      *  across the list), or NIL if there are no usable conditions.
      *  返回CTID条件表达式链表,如无则返回NIL
      *
      *  If the expression is an AND clause, we can use a CTID condition
      *  from any sub-clause.  If it is an OR clause, we must be able to
      *  extract a CTID condition from every sub-clause, or we can't use it.
      *  如为AND子句,从任意一个sub-clause中获取;如为OR则从每一个sub-clause中获取
      *
      *  In theory, in the AND case we could get CTID conditions from different
      *  sub-clauses, in which case we could try to pick the most efficient one.
      *  In practice, such usage seems very unlikely, so we don't bother; we
      *  just exit as soon as we find the first candidate.
      */
     static List *
     TidQualFromExpr(Node *expr, int varno)
     {
         List       *rlst = NIL;
         ListCell   *l;
     
         if (is_opclause(expr))//常规的表达式
         {
             /* base case: check for tideq opclause */
             if (IsTidEqualClause((OpExpr *) expr, varno))
                 rlst = list_make1(expr);
         }
         else if (expr && IsA(expr, ScalarArrayOpExpr))//ScalarArrayOpExpr
         {
             /* another base case: check for tid = ANY clause */
             if (IsTidEqualAnyClause((ScalarArrayOpExpr *) expr, varno))
                 rlst = list_make1(expr);
         }
         else if (expr && IsA(expr, CurrentOfExpr))//CurrentOfExpr
         {
             /* another base case: check for CURRENT OF on this rel */
             if (((CurrentOfExpr *) expr)->cvarno == varno)
                 rlst = list_make1(expr);
         }
         else if (and_clause(expr))//AND
         {
             foreach(l, ((BoolExpr *) expr)->args)
             {
                 rlst = TidQualFromExpr((Node *) lfirst(l), varno);
                 if (rlst)
                     break;
             }
         }
         else if (or_clause(expr))//OR
         {
             foreach(l, ((BoolExpr *) expr)->args)
             {
                 List       *frtn = TidQualFromExpr((Node *) lfirst(l), varno);
     
                 if (frtn)
                     rlst = list_concat(rlst, frtn);
                 else
                 {
                     if (rlst)
                         list_free(rlst);
                     rlst = NIL;
                     break;
                 }
             }
         }
         return rlst;
     }
    
    
    //-------------------------------------------------------------------------- create_tidscan_path
    
    /*
     * create_tidscan_path
     *    Creates a path corresponding to a scan by TID, returning the pathnode.
     *    创建访问路径,返回TidPath
     */
    TidPath *
    create_tidscan_path(PlannerInfo *root, RelOptInfo *rel, List *tidquals,
                        Relids required_outer)
    {
        TidPath    *pathnode = makeNode(TidPath);
    
        pathnode->path.pathtype = T_TidScan;
        pathnode->path.parent = rel;
        pathnode->path.pathtarget = rel->reltarget;
        pathnode->path.param_info = get_baserel_parampathinfo(root, rel,
                                                              required_outer);
        pathnode->path.parallel_aware = false;
        pathnode->path.parallel_safe = rel->consider_parallel;
        pathnode->path.parallel_workers = 0;
        pathnode->path.pathkeys = NIL;  /* always unordered */
    
        pathnode->tidquals = tidquals;
    
        cost_tidscan(&pathnode->path, root, rel, tidquals,
                     pathnode->path.param_info);//计算成本
    
        return pathnode;
    }
    
    //-------------------------------------------------------- cost_tidscan
    
     /*
      * cost_tidscan
      *    Determines and returns the cost of scanning a relation using TIDs.
      *    计算并返回使用TIDs扫描的成本
      *
      * 'baserel' is the relation to be scanned
      * baserel-基础关系
      * 'tidquals' is the list of TID-checkable quals
      * tidquals-TID条件表达式
      * 'param_info' is the ParamPathInfo if this is a parameterized path, else NULL
      * param_info-参数化路径,如无则为NULL
      */
     void
     cost_tidscan(Path *path, PlannerInfo *root,
                  RelOptInfo *baserel, List *tidquals, ParamPathInfo *param_info)
     {
         Cost        startup_cost = 0;
         Cost        run_cost = 0;
         bool        isCurrentOf = false;
         QualCost    qpqual_cost;
         Cost        cpu_per_tuple;
         QualCost    tid_qual_cost;
         int         ntuples;
         ListCell   *l;
         double      spc_random_page_cost;
     
         /* Should only be applied to base relations */
         Assert(baserel->relid > 0);
         Assert(baserel->rtekind == RTE_RELATION);
     
         /* Mark the path with the correct row estimate */
         if (param_info)
             path->rows = param_info->ppi_rows;
         else
             path->rows = baserel->rows;//行数
     
         /* Count how many tuples we expect to retrieve */
         ntuples = 0;
         foreach(l, tidquals)//遍历条件表达式
         {
             if (IsA(lfirst(l), ScalarArrayOpExpr))//ScalarArrayOpExpr
             {
                 /* Each element of the array yields 1 tuple */
                 ScalarArrayOpExpr *saop = (ScalarArrayOpExpr *) lfirst(l);
                 Node       *arraynode = (Node *) lsecond(saop->args);
     
                 ntuples += estimate_array_length(arraynode);
             }
             else if (IsA(lfirst(l), CurrentOfExpr))//CurrentOfExpr
             {
                 /* CURRENT OF yields 1 tuple */
                 isCurrentOf = true;
                 ntuples++;
             }
             else
             {
                 /* It's just CTID = something, count 1 tuple */
                 ntuples++;//计数+1
             }
         }
     
         /*
          * We must force TID scan for WHERE CURRENT OF, because only nodeTidscan.c
          * understands how to do it correctly.  Therefore, honor enable_tidscan
          * only when CURRENT OF isn't present.  Also note that cost_qual_eval
          * counts a CurrentOfExpr as having startup cost disable_cost, which we
          * subtract off here; that's to prevent other plan types such as seqscan
          * from winning.
          */
         if (isCurrentOf)//CurrentOfExpr
         {
             Assert(baserel->baserestrictcost.startup >= disable_cost);
             startup_cost -= disable_cost;
         }
         else if (!enable_tidscan)//如禁用tidscan
             startup_cost += disable_cost;//设置为高成本
     
         /*
          * The TID qual expressions will be computed once, any other baserestrict
          * quals once per retrieved tuple.
          * TID条件表达式每计算一次，其他基本类型的表达式亦计算一次
          */
         cost_qual_eval(&tid_qual_cost, tidquals, root);
     
         /* fetch estimated page cost for tablespace containing table */
         get_tablespace_page_costs(baserel->reltablespace,
                                   &spc_random_page_cost,
                                   NULL);//表空间page访问成本
     
         /* IO成本,假定每个元组都在不同的page中.disk costs --- assume each tuple on a different page */
         run_cost += spc_random_page_cost * ntuples;//运行成本
     
         /* CPU成本,Add scanning CPU costs */
         get_restriction_qual_cost(root, baserel, param_info, &qpqual_cost);//CPU扫描成本
     
         /* XXX currently we assume TID quals are a subset of qpquals */
         startup_cost += qpqual_cost.startup + tid_qual_cost.per_tuple;
         cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple -             tid_qual_cost.per_tuple;
         run_cost += cpu_per_tuple * ntuples;
     
         /* tlist eval costs are paid per output row, not per tuple scanned */
         startup_cost += path->pathtarget->cost.startup;
         run_cost += path->pathtarget->cost.per_tuple * path->rows;
     
         path->startup_cost = startup_cost;
         path->total_cost = startup_cost + run_cost;
     }
     
    

### 三、跟踪分析

测试脚本如下

    
    
    select a.ctid,a.dwbh,a.dwmc,b.grbh,b.xm,b.xb,b.nl 
    from t_dwxx a,t_grxx b 
    where a.ctid = '(2,10)'::tid 
          and a.dwbh = b.dwbh;
    
    

启动gdb,设置断点

    
    
    (gdb) b create_tidscan_paths
    Breakpoint 2 at 0x759b06: file tidpath.c, line 263.
    (gdb) c
    Continuing.
    
    Breakpoint 2, create_tidscan_paths (root=0x2869588, rel=0x2869998) at tidpath.c:263
    263     required_outer = rel->lateral_relids;
    

进入create_tidscan_paths->TidQualFromBaseRestrictinfo函数

    
    
    (gdb) n
    265     tidquals = TidQualFromBaseRestrictinfo(rel);
    (gdb) step
    TidQualFromBaseRestrictinfo (rel=0x2869998) at tidpath.c:225
    225     List       *rlst = NIL;
    

获取TID条件表达式,对应的是:a.ctid = '(2,10)'::tid

    
    
    ...
    (gdb) p *(Var *)$tmp->args->head->data.ptr_value
    $11 = {xpr = {type = T_Var}, varno = 1, varattno = -1, vartype = 27, vartypmod = -1, varcollid = 0, varlevelsup = 0, 
      varnoold = 1, varoattno = -1, location = 81}
    (gdb) p *(Const *)$tmp->args->head->next->data.ptr_value
    $12 = {xpr = {type = T_Const}, consttype = 27, consttypmod = -1, constcollid = 0, constlen = 6, constvalue = 41705832, 
      constisnull = false, constbyval = false, location = 90}
    

进入create_tidscan_path函数

    
    
    (gdb) 
    create_tidscan_paths (root=0x2869588, rel=0x2869998) at tidpath.c:267
    267     if (tidquals)
    (gdb) n
    268         add_path(rel, (Path *) create_tidscan_path(root, rel, tidquals,
    (gdb) step
    create_tidscan_path (root=0x2869588, rel=0x2869998, tidquals=0x287ef90, required_outer=0x0) at pathnode.c:1191
    1191        TidPath    *pathnode = makeNode(TidPath);
    

进入cost_tidscan

    
    
    (gdb) step
    cost_tidscan (path=0x287eee0, root=0x2869588, baserel=0x2869998, tidquals=0x287ef90, param_info=0x0) at costsize.c:1184
    1184        Cost        startup_cost = 0;
    #解析表达式的CPU成本
    (gdb) 
    1249        cost_qual_eval(&tid_qual_cost, tidquals, root);
    (gdb) 
    1252        get_tablespace_page_costs(baserel->reltablespace,
    (gdb) p tid_qual_cost
    $14 = {startup = 0, per_tuple = 0.0025000000000000001}
    

计算完毕,返回结果

    
    
    ...
    (gdb) 
    1272        path->startup_cost = startup_cost;
    (gdb) 
    1273        path->total_cost = startup_cost + run_cost;
    (gdb) 
    1274    }
    (gdb) 
    (gdb) p *path
    $17 = {type = T_TidPath, pathtype = T_TidScan, parent = 0x2869998, pathtarget = 0x287ac38, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 1, startup_cost = 0.0025000000000000001, 
      total_cost = 4.0125000000000002, pathkeys = 0x0}
    

结束create_tidscan_paths函数调用

    
    
    (gdb) n
    create_tidscan_path (root=0x2869588, rel=0x2869998, tidquals=0x287ef90, required_outer=0x0) at pathnode.c:1208
    1208        return pathnode;
    (gdb) 
    1209    }
    (gdb) 
    create_tidscan_paths (root=0x2869588, rel=0x2869998) at tidpath.c:270
    270 }
    (gdb) 
    set_plain_rel_pathlist (root=0x2869588, rel=0x2869998, rte=0x27c5318) at allpaths.c:718
    718 }
    

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

