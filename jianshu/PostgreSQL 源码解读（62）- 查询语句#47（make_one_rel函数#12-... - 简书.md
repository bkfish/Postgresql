本节大体介绍了make_one_rel函数中的make_rel_from_joinlist函数，该函数根据连接关系链表（joinlist）构建连接路径。

### 一、源码解读

make_rel_from_joinlist函数根据连接关系链表（joinlist）通过外部算法(钩子函数)/遗传算法/动态规划算法构建连接路径,其中joinlist链表在主函数中已通过调用deconstruct_jointree函数生成.  
动态规划算法的实现standard_join_search函数以及遗传算法在后续章节再行介绍.

    
    
     /*
      * make_rel_from_joinlist
      *    Build access paths using a "joinlist" to guide the join path search.
      *    依据deconstruct_jointree函数构造的joinlist生成连接路径.
      *    joinlist详细的数据结构参照deconstruct_jointree函数注释
      *
      * See comments for deconstruct_jointree() for definition of the joinlist
      * data structure.
      */
     static RelOptInfo *
     make_rel_from_joinlist(PlannerInfo *root, List *joinlist)
     {
         int         levels_needed;
         List       *initial_rels;
         ListCell   *jl;
     
         /*
          * Count the number of child joinlist nodes.  This is the depth of the
          * dynamic-programming algorithm we must employ to consider all ways of
          * joining the child nodes.
          * 计算joinlist链表中节点的个数。
          * 确定使用的算法(动态规划算法 vs 遗传算法),如个数<阈值,则考虑所有连接的方式。
          */
         levels_needed = list_length(joinlist);
     
         if (levels_needed <= 0)
             return NULL;            /* nothing to do? */
     
         /*
          * Construct a list of rels corresponding to the child joinlist nodes.
          * This may contain both base rels and rels constructed according to
          * sub-joinlists.
          * 构造与joinlist中元素相对应的rels链表。
          * 这可能包括base rels和通过子连接构造的base rels。
          */
         initial_rels = NIL;
         foreach(jl, joinlist)//遍历链表
         {
             Node       *jlnode = (Node *) lfirst(jl);
             RelOptInfo *thisrel;
     
             if (IsA(jlnode, RangeTblRef))//RTR
             {
                 int         varno = ((RangeTblRef *) jlnode)->rtindex;
     
                 thisrel = find_base_rel(root, varno);//根据编号找到相应的RelOptInfo
             }
             else if (IsA(jlnode, List))//链表
             {
                 /* Recurse to handle subproblem */
                 thisrel = make_rel_from_joinlist(root, (List *) jlnode);//递归调用,形成新的base rel
             }
             else//其他类型,出错
             {
                 elog(ERROR, "unrecognized joinlist node type: %d",
                      (int) nodeTag(jlnode));
                 thisrel = NULL;     /* keep compiler quiet */
             }
     
             initial_rels = lappend(initial_rels, thisrel);//添加到base rel链表中
         }
     
         if (levels_needed == 1)//连接链表只有1个元素
         {
             /*
              * Single joinlist node, so we're done.
              */
             return (RelOptInfo *) linitial(initial_rels);//直接返回
         }
         else//>1个元素
         {
             /*
              * Consider the different orders in which we could join the rels,
              * using a plugin, GEQO, or the regular join search code.
              * 考虑不同的连接顺序->使用外部算法/GEQO遗传算法/动态规划算法。
              *
              * We put the initial_rels list into a PlannerInfo field because
              * has_legal_joinclause() needs to look at it (ugly :-().
              *
              */
             root->initial_rels = initial_rels;
     
             if (join_search_hook)//调用钩子函数
                 return (*join_search_hook) (root, levels_needed, initial_rels);
             else if (enable_geqo && levels_needed >= geqo_threshold)
                 return geqo(root, levels_needed, initial_rels);//遗传算法
             else
                 return standard_join_search(root, levels_needed, initial_rels);//动态规划算法
         }
     }
     
    //----------------------------------------------------------------------- standard_join_search
     /*
      * standard_join_search
      *    Find possible joinpaths for a query by successively finding ways
      *    to join component relations into join relations.
      *    通过动态规划算法为查询语句构造连接路径.
      *
      * 'levels_needed' is the number of iterations needed, ie, the number of
      *      independent jointree items in the query.  This is > 1.
      * levels_needed-连接链表中的节点个数,>1
      *
      * 'initial_rels' is a list of RelOptInfo nodes for each independent
      *      jointree item.  These are the components to be joined together.
      *      Note that levels_needed == list_length(initial_rels).
      * initial_rels-与连接树每个元素相对应的RelOptInfo节点
      *
      * Returns the final level of join relations, i.e., the relation that is
      * the result of joining all the original relations together.
      * At least one implementation path must be provided for this relation and
      * all required sub-relations.
      * 返回连接的最终关系(最顶层的Relation):将所有原始关系连接在一起的最终结果。
      * 优化器为该关系及其所必需的子关系提供至少一个的实现路径。
      *
      * To support loadable plugins that modify planner behavior by changing the
      * join searching algorithm, we provide a hook variable that lets a plugin
      * replace or supplement this function.  Any such hook must return the same
      * final join relation as the standard code would, but it might have a
      * different set of implementation paths attached, and only the sub-joinrels
      * needed for these paths need have been instantiated.
      * 为了支持自定义函数，PG提供了一个钩子变量，允许外部插件替换或填充这个函数。
      * 任何这样的钩子都必须返回与PG标准函数相同的最终连接关系，
      * 但是它可能附加了一组不同的实现路径，并且只实例化了这些路径所需的子连接。
      *
      * Note to plugin authors: the functions invoked during standard_join_search()
      * modify root->join_rel_list and root->join_rel_hash.  If you want to do more
      * than one join-order search, you'll probably need to save and restore the
      * original states of those data structures.  See geqo_eval() for an example.
      */
     RelOptInfo *
     standard_join_search(PlannerInfo *root, int levels_needed, List *initial_rels)
     {
         int         lev;
         RelOptInfo *rel;
     
         /*
          * This function cannot be invoked recursively within any one planning
          * problem, so join_rel_level[] can't be in use already.
          */
         Assert(root->join_rel_level == NULL);//验证
     
         /*
          * We employ a simple "dynamic programming" algorithm: we first find all
          * ways to build joins of two jointree items, then all ways to build joins
          * of three items (from two-item joins and single items), then four-item
          * joins, and so on until we have considered all ways to join all the
          * items into one rel.
          * PG实现了一种简单的动态规划算法:首先为连接树中的两个Relation建立可能的连接路径
          * 然后为三个Relation建立所有可能的连接路径,以此类推直至已为所有的Relation建立了
          * 连接路径,从而得到最终的关系(final_rel)
          * 
          * root->join_rel_level[j] is a list of all the j-item rels.  Initially we
          * set root->join_rel_level[1] to represent all the single-jointree-item
          * relations.
          * 设置root->join_rel_level数组,[j]是所有j-item rels的链表(即1个item的放在[1]中)
          */
         root->join_rel_level = (List **) palloc0((levels_needed + 1) * sizeof(List *));
     
         root->join_rel_level[1] = initial_rels;//1个item对应的rel链表
     
         for (lev = 2; lev <= levels_needed; lev++)//构造2->N个item对应的rel链表
         {
             ListCell   *lc;
     
             /*
              * Determine all possible pairs of relations to be joined at this
              * level, and build paths for making each one from every available
              * pair of lower-level relations.
              * 确定在此级别上要连接的所有可能的关系对，并构建访问路径，
              * 以从每一对可用的较低级关系中往上创建关系。
              */
             join_search_one_level(root, lev);
     
             /*
              * Run generate_partitionwise_join_paths() and generate_gather_paths()
              * for each just-processed joinrel.  We could not do this earlier
              * because both regular and partial paths can get added to a
              * particular joinrel at multiple times within join_search_one_level.
              * 循环调用generate_partitionwise_join_paths()和generate_collect _paths()函数:
              * 参数为上一步骤生成的链表中的每个元素。
              * 由于常规路径和部分路径都可以在join_search_one_level中多次添加joinrel,因此在此处调用。
              *
              * After that, we're done creating paths for the joinrel, so run
              * set_cheapest().
              * 在此之后,PG已为joinrel(连接生成的新关系)创建了访问路径,因此可以调用函数set_cheapest
              *
              */
             foreach(lc, root->join_rel_level[lev])//遍历链表
             {
                 rel = (RelOptInfo *) lfirst(lc);//新生成的关系
     
                 /* Create paths for partitionwise joins. */
                 generate_partitionwise_join_paths(root, rel);//创建partitionwise路径
     
                 /*
                  * Except for the topmost scan/join rel, consider gathering
                  * partial paths.  We'll do the same for the topmost scan/join rel
                  * once we know the final targetlist (see grouping_planner).
                  */
                 if (lev < levels_needed)
                     generate_gather_paths(root, rel, false);//并行执行需考虑gathering
     
                 /* Find and save the cheapest paths for this rel */
                 set_cheapest(rel);//从形成该joinrel的所有路径中找到成本最低的
     
     #ifdef OPTIMIZER_DEBUG
                 debug_print_rel(root, rel);//DEBUG信息
     #endif
             }
         }
     
         /*
          * We should have a single rel at the final level.
          * 连接的最终结果,只有一个RelOptInfo
          */
         if (root->join_rel_level[levels_needed] == NIL)
             elog(ERROR, "failed to build any %d-way joins", levels_needed);
         Assert(list_length(root->join_rel_level[levels_needed]) == 1);
     
         rel = (RelOptInfo *) linitial(root->join_rel_level[levels_needed]);//获取最终结果
     
         root->join_rel_level = NULL;//重置
     
         return rel;//返回
     }
    
    //----------------------------------------------------------------------- geqo
     /*
      * geqo
      *    solution of the query optimization problem
      *    similar to a constrained Traveling Salesman Problem (TSP)
      *    遗传算法:可参考TSP的求解算法.
      *    TSP-旅行推销员问题（最短路径问题）:
      *        给定一系列城市和每对城市之间的距离，求解访问每一座城市一次并回到起始城市的最短回路。
      */
     
     RelOptInfo *
     geqo(PlannerInfo *root, int number_of_rels, List *initial_rels)
     {
         GeqoPrivateData private;
         int         generation;
         Chromosome *momma;
         Chromosome *daddy;
         Chromosome *kid;
         Pool       *pool;
         int         pool_size,
                     number_generations;
     
     #ifdef GEQO_DEBUG
         int         status_interval;
     #endif
         Gene       *best_tour;
         RelOptInfo *best_rel;
     
     #if defined(ERX)
         Edge       *edge_table;     /* list of edges */
         int         edge_failures = 0;
     #endif
     #if defined(CX) || defined(PX) || defined(OX1) || defined(OX2)
         City       *city_table;     /* list of cities */
     #endif
     #if defined(CX)
         int         cycle_diffs = 0;
         int         mutations = 0;
     #endif
     
     /* set up private information */
         root->join_search_private = (void *) &private;
         private.initial_rels = initial_rels;
     
     /* initialize private number generator */
         geqo_set_seed(root, Geqo_seed);
     
     /* set GA parameters */
         pool_size = gimme_pool_size(number_of_rels);
         number_generations = gimme_number_generations(pool_size);
     #ifdef GEQO_DEBUG
         status_interval = 10;
     #endif
     
     /* allocate genetic pool memory */
         pool = alloc_pool(root, pool_size, number_of_rels);
     
     /* random initialization of the pool */
         random_init_pool(root, pool);
     
     /* sort the pool according to cheapest path as fitness */
         sort_pool(root, pool);      /* we have to do it only one time, since all
                                      * kids replace the worst individuals in
                                      * future (-> geqo_pool.c:spread_chromo ) */
     
     #ifdef GEQO_DEBUG
         elog(DEBUG1, "GEQO selected %d pool entries, best %.2f, worst %.2f",
              pool_size,
              pool->data[0].worth,
              pool->data[pool_size - 1].worth);
     #endif
     
     /* allocate chromosome momma and daddy memory */
         momma = alloc_chromo(root, pool->string_length);
         daddy = alloc_chromo(root, pool->string_length);
     
     #if defined (ERX)
     #ifdef GEQO_DEBUG
         elog(DEBUG2, "using edge recombination crossover [ERX]");
     #endif
     /* allocate edge table memory */
         edge_table = alloc_edge_table(root, pool->string_length);
     #elif defined(PMX)
     #ifdef GEQO_DEBUG
         elog(DEBUG2, "using partially matched crossover [PMX]");
     #endif
     /* allocate chromosome kid memory */
         kid = alloc_chromo(root, pool->string_length);
     #elif defined(CX)
     #ifdef GEQO_DEBUG
         elog(DEBUG2, "using cycle crossover [CX]");
     #endif
     /* allocate city table memory */
         kid = alloc_chromo(root, pool->string_length);
         city_table = alloc_city_table(root, pool->string_length);
     #elif defined(PX)
     #ifdef GEQO_DEBUG
         elog(DEBUG2, "using position crossover [PX]");
     #endif
     /* allocate city table memory */
         kid = alloc_chromo(root, pool->string_length);
         city_table = alloc_city_table(root, pool->string_length);
     #elif defined(OX1)
     #ifdef GEQO_DEBUG
         elog(DEBUG2, "using order crossover [OX1]");
     #endif
     /* allocate city table memory */
         kid = alloc_chromo(root, pool->string_length);
         city_table = alloc_city_table(root, pool->string_length);
     #elif defined(OX2)
     #ifdef GEQO_DEBUG
         elog(DEBUG2, "using order crossover [OX2]");
     #endif
     /* allocate city table memory */
         kid = alloc_chromo(root, pool->string_length);
         city_table = alloc_city_table(root, pool->string_length);
     #endif
     
     
     /* my pain main part: */
     /* iterative optimization */
     
         for (generation = 0; generation < number_generations; generation++)
         {
             /* SELECTION: using linear bias function */
             geqo_selection(root, momma, daddy, pool, Geqo_selection_bias);
     
     #if defined (ERX)
             /* EDGE RECOMBINATION CROSSOVER */
             gimme_edge_table(root, momma->string, daddy->string, pool->string_length, edge_table);
     
             kid = momma;
     
             /* are there any edge failures ? */
             edge_failures += gimme_tour(root, edge_table, kid->string, pool->string_length);
     #elif defined(PMX)
             /* PARTIALLY MATCHED CROSSOVER */
             pmx(root, momma->string, daddy->string, kid->string, pool->string_length);
     #elif defined(CX)
             /* CYCLE CROSSOVER */
             cycle_diffs = cx(root, momma->string, daddy->string, kid->string, pool->string_length, city_table);
             /* mutate the child */
             if (cycle_diffs == 0)
             {
                 mutations++;
                 geqo_mutation(root, kid->string, pool->string_length);
             }
     #elif defined(PX)
             /* POSITION CROSSOVER */
             px(root, momma->string, daddy->string, kid->string, pool->string_length, city_table);
     #elif defined(OX1)
             /* ORDER CROSSOVER */
             ox1(root, momma->string, daddy->string, kid->string, pool->string_length, city_table);
     #elif defined(OX2)
             /* ORDER CROSSOVER */
             ox2(root, momma->string, daddy->string, kid->string, pool->string_length, city_table);
     #endif
     
     
             /* EVALUATE FITNESS */
             kid->worth = geqo_eval(root, kid->string, pool->string_length);
     
             /* push the kid into the wilderness of life according to its worth */
             spread_chromo(root, kid, pool);
     
     
     #ifdef GEQO_DEBUG
             if (status_interval && !(generation % status_interval))
                 print_gen(stdout, pool, generation);
     #endif
     
         }
     
     
     #if defined(ERX) && defined(GEQO_DEBUG)
         if (edge_failures != 0)
             elog(LOG, "[GEQO] failures: %d, average: %d",
                  edge_failures, (int) number_generations / edge_failures);
         else
             elog(LOG, "[GEQO] no edge failures detected");
     #endif
     
     #if defined(CX) && defined(GEQO_DEBUG)
         if (mutations != 0)
             elog(LOG, "[GEQO] mutations: %d, generations: %d",
                  mutations, number_generations);
         else
             elog(LOG, "[GEQO] no mutations processed");
     #endif
     
     #ifdef GEQO_DEBUG
         print_pool(stdout, pool, 0, pool_size - 1);
     #endif
     
     #ifdef GEQO_DEBUG
         elog(DEBUG1, "GEQO best is %.2f after %d generations",
              pool->data[0].worth, number_generations);
     #endif
     
     
         /*
          * got the cheapest query tree processed by geqo; first element of the
          * population indicates the best query tree
          */
         best_tour = (Gene *) pool->data[0].string;
     
         best_rel = gimme_tree(root, best_tour, pool->string_length);
     
         if (best_rel == NULL)
             elog(ERROR, "geqo failed to make a valid plan");
     
         /* DBG: show the query plan */
     #ifdef NOT_USED
         print_plan(best_plan, root);
     #endif
     
         /* ... free memory stuff */
         free_chromo(root, momma);
         free_chromo(root, daddy);
     
     #if defined (ERX)
         free_edge_table(root, edge_table);
     #elif defined(PMX)
         free_chromo(root, kid);
     #elif defined(CX)
         free_chromo(root, kid);
         free_city_table(root, city_table);
     #elif defined(PX)
         free_chromo(root, kid);
         free_city_table(root, city_table);
     #elif defined(OX1)
         free_chromo(root, kid);
         free_city_table(root, city_table);
     #elif defined(OX2)
         free_chromo(root, kid);
         free_city_table(root, city_table);
     #endif
     
         free_pool(root, pool);
     
         /* ... clear root pointer to our private storage */
         root->join_search_private = NULL;
     
         return best_rel;
     }
    
    

### 二、跟踪分析

测试脚本以及执行计划如下:

    
    
    testdb=# explain verbose select a.*,b.grbh,b.je 
    testdb-# from t_dwxx a,
    testdb-#     lateral (select t1.dwbh,t1.grbh,t2.je 
    testdb(#      from t_grxx t1 
    testdb(#           inner join t_jfxx t2 on t1.dwbh = a.dwbh and t1.grbh = t2.grbh) b
    testdb-# where a.dwbh = '1001'
    testdb-# order by b.dwbh;
                                                  QUERY PLAN                                              
    ------------------------------------------------------------------------------------------------------     Nested Loop  (cost=0.87..111.89 rows=10 width=37)
       Output: a.dwmc, a.dwbh, a.dwdz, t1.grbh, t2.je, t1.dwbh
       ->  Nested Loop  (cost=0.58..28.69 rows=10 width=29)
             Output: a.dwmc, a.dwbh, a.dwdz, t1.grbh, t1.dwbh
             ->  Index Scan using t_dwxx_pkey on public.t_dwxx a  (cost=0.29..8.30 rows=1 width=20)
                   Output: a.dwmc, a.dwbh, a.dwdz
                   Index Cond: ((a.dwbh)::text = '1001'::text)
             ->  Index Scan using idx_t_grxx_dwbh on public.t_grxx t1  (cost=0.29..20.29 rows=10 width=9)
                   Output: t1.dwbh, t1.grbh, t1.xm, t1.xb, t1.nl
                   Index Cond: ((t1.dwbh)::text = '1001'::text)
       ->  Index Scan using idx_t_jfxx_grbh on public.t_jfxx t2  (cost=0.29..8.31 rows=1 width=13)
             Output: t2.grbh, t2.ny, t2.je
             Index Cond: ((t2.grbh)::text = (t1.grbh)::text)
    

启动gdb跟踪

    
    
    (gdb) b make_rel_from_joinlist
    Breakpoint 1 at 0x73f0d3: file allpaths.c, line 2617.
    (gdb) c
    Continuing.
    
    Breakpoint 1, make_rel_from_joinlist (root=0x176c750, joinlist=0x179e480) at allpaths.c:2617
    2617        levels_needed = list_length(joinlist);
    

进入make_rel_from_joinlist函数,查看joinlist,链表中的Node为RangeTblRef,rindex分别是1/3/4

    
    
    (gdb) p *joinlist
    $1 = {type = T_List, length = 3, head = 0x17a0448, tail = 0x17a0408}
    (gdb) p *(Node *)joinlist->head->data.ptr_value
    $2 = {type = T_RangeTblRef}
    (gdb) p *(RangeTblRef *)joinlist->head->data.ptr_value
    $3 = {type = T_RangeTblRef, rtindex = 1}
    (gdb) p *(RangeTblRef *)joinlist->head->next->data.ptr_value
    $4 = {type = T_RangeTblRef, rtindex = 3}
    (gdb) p *(RangeTblRef *)joinlist->head->next->next->data.ptr_value
    $5 = {type = T_RangeTblRef, rtindex = 4}
    

链表中的Node个数为3,levels_needed=3

    
    
    (gdb) n
    2619        if (levels_needed <= 0)
    (gdb) p levels_needed
    $6 = 3
    

遍历链表,构造RelOptInfo,添加到initial_rels中

    
    
    (gdb) 
    2628        foreach(jl, joinlist)
    ...
    (gdb) 
    2637                thisrel = find_base_rel(root, varno);
    (gdb) 
    2651            initial_rels = lappend(initial_rels, thisrel);
    

完成遍历后,开始构造连接路径.  
遗传算法的rels阈值为12(通过GUC参数配置)

    
    
    2672            if (join_search_hook)
    (gdb) 
    2674            else if (enable_geqo && levels_needed >= geqo_threshold)
    (gdb) 
    2677                return standard_join_search(root, levels_needed, initial_rels);
    (gdb) p geqo_threshold
    $7 = 12
    

进入函数standard_join_search

    
    
    (gdb) step
    standard_join_search (root=0x176c750, levels_needed=3, initial_rels=0x17a6308) at allpaths.c:2733
    2733        root->join_rel_level = (List **) palloc0((levels_needed + 1) * sizeof(List *));
    

开始构造2->N个item对应的rel链表

    
    
    ...
    (gdb) 
    2746            join_search_one_level(root, lev);
    (gdb) n
    2757            foreach(lc, root->join_rel_level[lev])
    

调用函数join_search_one_level,查看root->join_rel_level[j]

    
    
    (gdb) p *root->join_rel_level[2]
    $10 = {type = T_List, length = 2, head = 0x17a67a8, tail = 0x17a6ec0}
    

查看链表中的RelOptInfo

    
    
    (gdb) p *(RelOptInfo *)root->join_rel_level[2]->head->data.ptr_value
    $12 = {type = T_RelOptInfo, reloptkind = RELOPT_JOINREL, relids = 0x17a65d0, rows = 10, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x17a65e8, pathlist = 0x17a68a8, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x0, cheapest_total_path = 0x0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 0, reltablespace = 0, 
      rtekind = RTE_JOIN, min_attr = 0, max_attr = 0, attr_needed = 0x0, attr_widths = 0x0, lateral_vars = 0x0, 
      lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 0, tuples = 0, allvisfrac = 0, subroot = 0x0, 
      subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, fdwroutine = 0x0, 
      fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, baserestrictcost = {
        startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, has_eclass_joins = true, 
      top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, partition_qual = 0x0, part_rels = 0x0, 
      partexprs = 0x0, nullable_partexprs = 0x0, partitioned_child_rels = 0x0}
    (gdb) p *(RelOptInfo *)root->join_rel_level[2]->head->next->data.ptr_value
    $13 = {type = T_RelOptInfo, reloptkind = RELOPT_JOINREL, relids = 0x17a68d8, rows = 10, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x17a6cd0, pathlist = 0x17a7720, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x0, cheapest_total_path = 0x0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 0, reltablespace = 0, 
      rtekind = RTE_JOIN, min_attr = 0, max_attr = 0, attr_needed = 0x0, attr_widths = 0x0, lateral_vars = 0x0, 
      lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 0, tuples = 0, allvisfrac = 0, subroot = 0x0, 
      subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, fdwroutine = 0x0, 
      fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, baserestrictcost = {
        startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, has_eclass_joins = true, 
      top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, partition_qual = 0x0, part_rels = 0x0, 
      partexprs = 0x0, nullable_partexprs = 0x0, partitioned_child_rels = 0x0}
    

查看RelOptInfo中的relids  
通过relids可知,第一个RelOptInfo是1/3号rel连接生成的Relation,第二个RelOptInfo是3/4号rel连接生成的Relation

    
    
    (gdb) set $roi1=(RelOptInfo *)root->join_rel_level[2]->head->data.ptr_value
    (gdb) set $roi2=(RelOptInfo *)root->join_rel_level[2]->head->next->data.ptr_value
    (gdb) p *$roi1->relids
    $16 = {nwords = 1, words = 0x17a65d4}
    (gdb) p *$roi1->relids->words
    $17 = 10  -->2 + 8 --> 1/3 号rel
    (gdb) p *$roi2->relids->words
    $18 = 24  -->8 + 16 --> 3/4号rel
    

查看第一个RelOptInfo中的pathlist,该链表有2个Node,类型均为T_NestPath(嵌套连接),总成本分别是28.69和4308.57

    
    
    (gdb) p *$roi1->pathlist
    $19 = {type = T_List, length = 2, head = 0x17a6888, tail = 0x17a6a10}
    (gdb) p *(Node *)$roi1->pathlist->head->data.ptr_value
    $20 = {type = T_NestPath}
    (gdb) p *(NestPath *)$roi1->pathlist->head->data.ptr_value
    $21 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x17a63c0, pathtarget = 0x17a65e8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.57750000000000001, 
        total_cost = 28.688484322533327, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x17a2638, innerjoinpath = 0x17a2908, joinrestrictinfo = 0x0}
    (gdb) p *(Node *)$roi1->pathlist->head->next->data.ptr_value
    $22 = {type = T_NestPath}
    (gdb) p *(NestPath *)$roi1->pathlist->head->next->data.ptr_value
    $23 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x17a63c0, pathtarget = 0x17a65e8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.57750000000000001, 
        total_cost = 4308.5748727883229, pathkeys = 0x17a3650}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x17a3190, innerjoinpath = 0x17a68f0, joinrestrictinfo = 0x0}
    

查看第二个RelOptInfo中的pathlist,只有1个Node,类型为T_NestPath(嵌套连接),总成本为103.49

    
    
    (gdb) p *$roi2->pathlist
    $24 = {type = T_List, length = 1, head = 0x17a7700, tail = 0x17a7700}
    (gdb) p *(Node *)$roi2->pathlist->head->data.ptr_value
    $27 = {type = T_NestPath}
    (gdb) p *(NestPath *)$roi2->pathlist->head->data.ptr_value
    $28 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x17a6ac0, pathtarget = 0x17a6cd0, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.58499999999999996, 
        total_cost = 103.48598432253331, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x17a2908, innerjoinpath = 0x17a5470, joinrestrictinfo = 0x0}
    

通过set_cheapest函数设置成本最低的访问路径,结果存储在cheapest_startup_path和cheapest_total_path中

    
    
    (gdb) 
    2773                set_cheapest(rel);
    (gdb) 
    2757            foreach(lc, root->join_rel_level[lev])
    ...
    (gdb) p *$roi1
    $35 = ..., cheapest_startup_path = 0x17a67f8, cheapest_total_path = 0x17a67f8, ...
    (gdb) p *$roi2
    $36 =..., cheapest_startup_path = 0x17a7750, cheapest_total_path = 0x17a7750, ...
    

继续循环,这时候lev=3

    
    
    (gdb) n
    2737        for (lev = 2; lev <= levels_needed; lev++)
    (gdb) n
    2746            join_search_one_level(root, lev);
    (gdb) p lev
    $38 = 3
    

得到3张表连接的final_rel

    
    
    (gdb) p *root->join_rel_level[3]
    $41 = {type = T_List, length = 1, head = 0x17a8090, tail = 0x17a8090}
    (gdb) p *(RelOptInfo *)root->join_rel_level[3]->head->data.ptr_value
    $42 = {type = T_RelOptInfo, reloptkind = RELOPT_JOINREL, relids = 0x17a74d8, rows = 10, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x17a7e40, pathlist = 0x17a8258, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x0, cheapest_total_path = 0x0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 0, reltablespace = 0, 
      rtekind = RTE_JOIN, min_attr = 0, max_attr = 0, attr_needed = 0x0, attr_widths = 0x0, lateral_vars = 0x0, 
      lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 0, tuples = 0, allvisfrac = 0, subroot = 0x0, 
      subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, fdwroutine = 0x0, 
      fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, baserestrictcost = {
        startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, has_eclass_joins = false, 
      top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, partition_qual = 0x0, part_rels = 0x0, 
      partexprs = 0x0, nullable_partexprs = 0x0, partitioned_child_rels = 0x0}
    

查看pathlist,只有1个元素,类型为NestPath,该访问路径成本为111.89

    
    
    (gdb) set $roi=(RelOptInfo *)root->join_rel_level[3]->head->data.ptr_value
    (gdb) p *$roi->pathlist
    $44 = {type = T_List, length = 1, head = 0x17a8238, tail = 0x17a8238}
    (gdb) p *(Node *)$roi->pathlist->head->data.ptr_value
    $45 = {type = T_NestPath}
    (gdb) p *(NestPath *)$roi->pathlist->head->data.ptr_value
    $46 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x17a7c30, pathtarget = 0x17a7e40, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.87, 
        total_cost = 111.88848432253332, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x17a67f8, innerjoinpath = 0x17a5470, joinrestrictinfo = 0x0}
    
    

获得最终结果

    
    
    ...
    2792        return rel;
    (gdb) p *rel
    $47 = {type = T_RelOptInfo, reloptkind = RELOPT_JOINREL, relids = 0x17a74d8, rows = 10, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x17a7e40, pathlist = 0x17a8258, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x17a8318, cheapest_total_path = 0x17a8318, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x17a89b0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 0, 
      reltablespace = 0, rtekind = RTE_JOIN, min_attr = 0, max_attr = 0, attr_needed = 0x0, attr_widths = 0x0, 
      lateral_vars = 0x0, lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 0, tuples = 0, allvisfrac = 0, 
      subroot = 0x0, subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, 
      fdwroutine = 0x0, fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, 
      baserestrictcost = {startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, 
      has_eclass_joins = false, top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, partition_qual = 0x0, 
      part_rels = 0x0, partexprs = 0x0, nullable_partexprs = 0x0, partitioned_child_rels = 0x0}
    (gdb) p *rel->cheapest_total_path
    $48 = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x17a7c30, pathtarget = 0x17a7e40, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 10, startup_cost = 0.87, 
      total_cost = 111.88848432253332, pathkeys = 0x0}  
    

DONE!

### 三、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

