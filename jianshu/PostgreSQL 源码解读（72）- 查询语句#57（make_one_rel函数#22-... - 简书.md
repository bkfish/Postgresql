本节大体介绍了遗传算法(geqo函数)的实现，在参与连接的关系大于等于12(默认值)个时，PG使用遗传算法生成连接访问路径，构建最终的连接关系。  
**遗传算法简介**  
遗传算法是借鉴生物科学而产生的搜索算法，在这个算法中会用到一些生物科学的相关知识，下面是PG遗传算法中所使用的的一些术语：  
1、染色体(Chromosome)：染色体又可称为基因型个体(individuals)，一个染色体可以视为一个解(一个合法的连接访问路径)。  
2、种群（Pool）：一定数量的个体（染色体）组成了群体(pool/population)，群体中个体的数量叫做群体大小（population size）。  
3、基因(Gene)：基因是染色体中的元素，用于表示个体的特征。例如有一个串（即染色体）S=1011，则其中的1，0，1，1这4个元素分别称为基因。在PG中，基因是参与连接的关系。  
4、适应度(Fitness)：各个个体对环境的适应程度叫做适应度(fitness)。为了体现染色体的适应能力，引入了对问题中的每一个染色体都能进行度量的函数，叫适应度函数。这个函数通常会被用来计算个体在群体中被使用的概率。在PG中适应度是连接访问路径的总成本。

### 一、数据结构

    
    
    /*
     * Private state for a GEQO run --- accessible via root->join_search_private
     */
    typedef struct
    {
        List       *initial_rels;   /* 参与连接的关系链表;the base relations we are joining */
        unsigned short random_state[3]; /* 无符号短整型数组(随机数);state for pg_erand48() */
    } GeqoPrivateData;
    
    /* we presume that int instead of Relid
       is o.k. for Gene; so don't change it! */
    typedef int Gene;//基因(整型)
    
    typedef struct Chromosome//染色体
    {
        Gene       *string;//基因
        Cost        worth;//成本
    } Chromosome;
    
    typedef struct Pool//种群
    {
        Chromosome *data;//染色体数组
        int         size;//大小
        int         string_length;//长度
    } Pool;
     
     /* A "clump" of already-joined relations within gimme_tree */
     typedef struct
     {
         RelOptInfo *joinrel;        /* joinrel for the set of relations */
         int         size;           /* number of input relations in clump */
     } Clump;
    

### 二、源码解读

geqo函数实现了遗传算法，构建多表（≥12）的连接访问路径。

    
    
    //----------------------------------------------------------------------- geqo
    /*
     * geqo
     *    solution of the query optimization problem
     *    similar to a constrained Traveling Salesman Problem (TSP)
     *     遗传算法:可参考TSP的求解算法.
     *    TSP-旅行推销员问题（最短路径问题）:
     *       给定一系列城市和每对城市之间的距离，求解访问每一座城市一次并回到起始城市的最短回路。
     */
    
    RelOptInfo *
    geqo(PlannerInfo *root, int number_of_rels, List *initial_rels)
    {
        GeqoPrivateData private;//遗传算法私有的数据,包括参与连接的关系和随机数
        int         generation;
        Chromosome *momma;//染色体-母亲数组
        Chromosome *daddy;//染色体-父亲数组
        Chromosome *kid;//染色体-孩子数组
        Pool       *pool;//种群指针
        int         pool_size,//种群大小
                    number_generations;//进化代数,使用最大迭代次数(进化代数)作为停止准则
    
    #ifdef GEQO_DEBUG
        int         status_interval;
    #endif
        Gene       *best_tour;
        RelOptInfo *best_rel;//最优解
    
    #if defined(ERX)
        Edge       *edge_table;     /* 边界链表;list of edges */
        int         edge_failures = 0;
    #endif
    #if defined(CX) || defined(PX) || defined(OX1) || defined(OX2)
        City       *city_table;     /* 城市链表;list of cities */
    #endif
    #if defined(CX)
        int         cycle_diffs = 0;
        int         mutations = 0;
    #endif
    
    /* 配置私有信息;set up private information */
        root->join_search_private = (void *) &private;
        private.initial_rels = initial_rels;
    
    /* 初始化种子值;initialize private number generator */
        geqo_set_seed(root, Geqo_seed);
    
    /* 设置遗传算法参数;set GA parameters */
        pool_size = gimme_pool_size(number_of_rels);//种群大小
        number_generations = gimme_number_generations(pool_size);//迭代次数
    #ifdef GEQO_DEBUG
        status_interval = 10;
    #endif
    
    /* 申请内存;allocate genetic pool memory */
        pool = alloc_pool(root, pool_size, number_of_rels);
    
    /* 随机初始化种群;random initialization of the pool */
        random_init_pool(root, pool);
    
    /* 对种群进行排序,成本最低的保留;sort the pool according to cheapest path as fitness */
        sort_pool(root, pool);      /* we have to do it only one time, since all
                                     * kids replace the worst individuals in
                                     * future (-> geqo_pool.c:spread_chromo ) */
    
    #ifdef GEQO_DEBUG
        elog(DEBUG1, "GEQO selected %d pool entries, best %.2f, worst %.2f",
             pool_size,
             pool->data[0].worth,
             pool->data[pool_size - 1].worth);
    #endif
    
    /* 申请染色体内存(母亲&父亲);allocate chromosome momma and daddy memory */
        momma = alloc_chromo(root, pool->string_length);
        daddy = alloc_chromo(root, pool->string_length);
    
    #if defined (ERX)
    #ifdef GEQO_DEBUG
        elog(DEBUG2, "using edge recombination crossover [ERX]");
    #endif
    /* allocate edge table memory */
      //申请边界表内存
        edge_table = alloc_edge_table(root, pool->string_length);
    #elif defined(PMX)
    #ifdef GEQO_DEBUG
        elog(DEBUG2, "using partially matched crossover [PMX]");
    #endif
    /* 申请孩子染色体内存;allocate chromosome kid memory */
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
        kid = alloc_chromo(root, pool->string_length);//申请内存
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
    /* 迭代式优化.iterative optimization */
    
        for (generation = 0; generation < number_generations; generation++)//开始迭代
        {
            /* SELECTION: using linear bias function */
        //选择:利用线性偏差(bias)函数,从中选出momma&daddy
            geqo_selection(root, momma, daddy, pool, Geqo_selection_bias);
    
    #if defined (ERX)
            /* EDGE RECOMBINATION CROSSOVER */
        //交叉遗传
            gimme_edge_table(root, momma->string, daddy->string, pool->string_length, edge_table);
    
            kid = momma;
    
            /* are there any edge failures ? */
        //遍历边界
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
        //计算适应度
            kid->worth = geqo_eval(root, kid->string, pool->string_length);
    
            /* push the kid into the wilderness of life according to its worth */
        //把遗传产生的染色体放到野外以进行下一轮的进化
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
    
    
    //--------------------------------------------------------------------------- geqo_pool.c
    static int  compare(const void *arg1, const void *arg2);
    
    /*
     * alloc_pool
     *      allocates memory for GA pool
     */
    Pool *
    alloc_pool(PlannerInfo *root, int pool_size, int string_length)
    {
        Pool       *new_pool;
        Chromosome *chromo;
        int         i;
    
        /* pool */
        new_pool = (Pool *) palloc(sizeof(Pool));
        new_pool->size = (int) pool_size;
        new_pool->string_length = (int) string_length;
    
        /* all chromosome */
        new_pool->data = (Chromosome *) palloc(pool_size * sizeof(Chromosome));
    
        /* all gene */
        chromo = (Chromosome *) new_pool->data; /* vector of all chromos */
        for (i = 0; i < pool_size; i++)
            chromo[i].string = palloc((string_length + 1) * sizeof(Gene));
    
        return new_pool;
    }
    
    /*
     * free_pool
     *      deallocates memory for GA pool
     */
    void
    free_pool(PlannerInfo *root, Pool *pool)
    {
        Chromosome *chromo;
        int         i;
    
        /* all gene */
        chromo = (Chromosome *) pool->data; /* vector of all chromos */
        for (i = 0; i < pool->size; i++)
            pfree(chromo[i].string);
    
        /* all chromosome */
        pfree(pool->data);
    
        /* pool */
        pfree(pool);
    }
    
    /*
     * random_init_pool
     *      initialize genetic pool
     */
    void
    random_init_pool(PlannerInfo *root, Pool *pool)
    {
        Chromosome *chromo = (Chromosome *) pool->data;
        int         i;
        int         bad = 0;
    
        /*
         * We immediately discard any invalid individuals (those that geqo_eval
         * returns DBL_MAX for), thereby not wasting pool space on them.
       * 立即丢弃所有无效的个体(那些geqo_eval返回DBL_MAX的)，因此不会在它们上浪费内存空间。
         *
         * If we fail to make any valid individuals after 10000 tries, give up;
         * this probably means something is broken, and we shouldn't just let
         * ourselves get stuck in an infinite loop.
       * 如果在10000次尝试后仍然没有产生任何有效的个体，那么放弃是最好的选择;
       * 这可能意味着有个地方存在问题，因此不应该陷入死循环。
         */
        i = 0;
        while (i < pool->size)
        {
            init_tour(root, chromo[i].string, pool->string_length);
            pool->data[i].worth = geqo_eval(root, chromo[i].string,
                                            pool->string_length);
            if (pool->data[i].worth < DBL_MAX)
                i++;
            else
            {
                bad++;
                if (i == 0 && bad >= 10000)
                    elog(ERROR, "geqo failed to make a valid plan");
            }
        }
    
    #ifdef GEQO_DEBUG
        if (bad > 0)
            elog(DEBUG1, "%d invalid tours found while selecting %d pool entries",
                 bad, pool->size);
    #endif
    }
    
    /*
     * sort_pool
     *   sorts input pool according to worth, from smallest to largest
     *
     *   maybe you have to change compare() for different ordering ...
     */
    void
    sort_pool(PlannerInfo *root, Pool *pool)
    {
        qsort(pool->data, pool->size, sizeof(Chromosome), compare);
    }
    
    /*
     * compare
     *   qsort comparison function for sort_pool
     */
    static int
    compare(const void *arg1, const void *arg2)
    {
        const Chromosome *chromo1 = (const Chromosome *) arg1;
        const Chromosome *chromo2 = (const Chromosome *) arg2;
    
        if (chromo1->worth == chromo2->worth)
            return 0;
        else if (chromo1->worth > chromo2->worth)
            return 1;
        else
            return -1;
    }
    
    /* alloc_chromo
     *    allocates a chromosome and string space
     */
    Chromosome *
    alloc_chromo(PlannerInfo *root, int string_length)
    {
        Chromosome *chromo;
    
        chromo = (Chromosome *) palloc(sizeof(Chromosome));
        chromo->string = (Gene *) palloc((string_length + 1) * sizeof(Gene));
    
        return chromo;
    }
    
    /* free_chromo
     *    deallocates a chromosome and string space
     */
    void
    free_chromo(PlannerInfo *root, Chromosome *chromo)
    {
        pfree(chromo->string);
        pfree(chromo);
    }
    
    /* spread_chromo
     *   inserts a new chromosome into the pool, displacing worst gene in pool
     *   assumes best->worst = smallest->largest
     */
    void
    spread_chromo(PlannerInfo *root, Chromosome *chromo, Pool *pool)
    {
        int         top,
                    mid,
                    bot;
        int         i,
                    index;
        Chromosome  swap_chromo,
                    tmp_chromo;
    
        /* new chromo is so bad we can't use it */
        if (chromo->worth > pool->data[pool->size - 1].worth)
            return;
    
        /* do a binary search to find the index of the new chromo */
    
        top = 0;
        mid = pool->size / 2;
        bot = pool->size - 1;
        index = -1;
    
        while (index == -1)
        {
            /* these 4 cases find a new location */
    
            if (chromo->worth <= pool->data[top].worth)
                index = top;
            else if (chromo->worth == pool->data[mid].worth)
                index = mid;
            else if (chromo->worth == pool->data[bot].worth)
                index = bot;
            else if (bot - top <= 1)
                index = bot;
    
    
            /*
             * these 2 cases move the search indices since a new location has not
             * yet been found.
             */
    
            else if (chromo->worth < pool->data[mid].worth)
            {
                bot = mid;
                mid = top + ((bot - top) / 2);
            }
            else
            {                       /* (chromo->worth > pool->data[mid].worth) */
                top = mid;
                mid = top + ((bot - top) / 2);
            }
        }                           /* ... while */
    
        /* now we have index for chromo */
    
        /*
         * move every gene from index on down one position to make room for chromo
         */
    
        /*
         * copy new gene into pool storage; always replace worst gene in pool
         */
    
        geqo_copy(root, &pool->data[pool->size - 1], chromo, pool->string_length);
    
        swap_chromo.string = pool->data[pool->size - 1].string;
        swap_chromo.worth = pool->data[pool->size - 1].worth;
    
        for (i = index; i < pool->size; i++)
        {
            tmp_chromo.string = pool->data[i].string;
            tmp_chromo.worth = pool->data[i].worth;
    
            pool->data[i].string = swap_chromo.string;
            pool->data[i].worth = swap_chromo.worth;
    
            swap_chromo.string = tmp_chromo.string;
            swap_chromo.worth = tmp_chromo.worth;
        }
    }
    
     /*
      * init_tour
      *
      *   Randomly generates a legal "traveling salesman" tour
      *   (i.e. where each point is visited only once.)
      *   随机生成TSP路径(每个点只访问一次)
      */
     void
     init_tour(PlannerInfo *root, Gene *tour, int num_gene)
     {
         int         i,
                     j;
     
         /*
          * We must fill the tour[] array with a random permutation of the numbers
          * 1 .. num_gene.  We can do that in one pass using the "inside-out"
          * variant of the Fisher-Yates shuffle algorithm.  Notionally, we append
          * each new value to the array and then swap it with a randomly-chosen
          * array element (possibly including itself, else we fail to generate
          * permutations with the last city last).  The swap step can be optimized
          * by combining it with the insertion.
          */
         if (num_gene > 0)
             tour[0] = (Gene) 1;
     
         for (i = 1; i < num_gene; i++)
         {
             j = geqo_randint(root, i, 0);
             /* i != j check avoids fetching uninitialized array element */
             if (i != j)
                 tour[i] = tour[j];
             tour[j] = (Gene) (i + 1);
         }
     }
     
    
    //----------------------------------------------------------- geqo_eval
     /*
      * geqo_eval
      *
      * Returns cost of a query tree as an individual of the population.
      * 返回该此进化的适应度。
      * 
      * If no legal join order can be extracted from the proposed tour,
      * returns DBL_MAX.
      * 如无合适的连接顺序,返回DBL_MAX
      */
     Cost
     geqo_eval(PlannerInfo *root, Gene *tour, int num_gene)
     {
         MemoryContext mycontext;
         MemoryContext oldcxt;
         RelOptInfo *joinrel;
         Cost        fitness;
         int         savelength;
         struct HTAB *savehash;
     
         /*
          * Create a private memory context that will hold all temp storage
          * allocated inside gimme_tree().
          *
          * Since geqo_eval() will be called many times, we can't afford to let all
          * that memory go unreclaimed until end of statement.  Note we make the
          * temp context a child of the planner's normal context, so that it will
          * be freed even if we abort via ereport(ERROR).
          */
         mycontext = AllocSetContextCreate(CurrentMemoryContext,
                                           "GEQO",
                                           ALLOCSET_DEFAULT_SIZES);
         oldcxt = MemoryContextSwitchTo(mycontext);
     
         /*
          * gimme_tree will add entries to root->join_rel_list, which may or may
          * not already contain some entries.  The newly added entries will be
          * recycled by the MemoryContextDelete below, so we must ensure that the
          * list is restored to its former state before exiting.  We can do this by
          * truncating the list to its original length.  NOTE this assumes that any
          * added entries are appended at the end!
          *
          * We also must take care not to mess up the outer join_rel_hash, if there
          * is one.  We can do this by just temporarily setting the link to NULL.
          * (If we are dealing with enough join rels, which we very likely are, a
          * new hash table will get built and used locally.)
          *
          * join_rel_level[] shouldn't be in use, so just Assert it isn't.
          */
         savelength = list_length(root->join_rel_list);
         savehash = root->join_rel_hash;
         Assert(root->join_rel_level == NULL);
     
         root->join_rel_hash = NULL;
     
         /* construct the best path for the given combination of relations */
       //给定的关系组合,构造最佳的访问路径
         joinrel = gimme_tree(root, tour, num_gene);
     
         /*
          * compute fitness, if we found a valid join
        * 如找到一个有效的连接,计算其适应度
          *
          * XXX geqo does not currently support optimization for partial result
          * retrieval, nor do we take any cognizance of possible use of
          * parameterized paths --- how to fix?
        * 遗传算法目前不支持部分结果检索的优化，目前也不知道是否可能使用参数化路径——如何修复?
          */
         if (joinrel)
         {
             Path       *best_path = joinrel->cheapest_total_path;//获取生成的关系的最优路径
     
             fitness = best_path->total_cost;//适应度=该路径的总成本
         }
         else
             fitness = DBL_MAX;//连接无效,适应度为DBL_MAX,下一轮迭代会丢弃
     
         /*
          * Restore join_rel_list to its former state, and put back original
          * hashtable if any.
        * 将join_rel_list恢复到原来的状态.如存在hash表，则把原来的哈希表放回去。
          */
         root->join_rel_list = list_truncate(root->join_rel_list,
                                             savelength);
         root->join_rel_hash = savehash;
     
         /* release all the memory acquired within gimme_tree */
       //释放资源
         MemoryContextSwitchTo(oldcxt);
         MemoryContextDelete(mycontext);
     
         return fitness;
     }
     
    //------------------------------------------- gimme_tree
    /*
     * gimme_tree
     *    Form planner estimates for a join tree constructed in the specified
     *    order.
     *    给定顺序构造连接树,由优化器估算成本. 
     *
     *   'tour' is the proposed join order, of length 'num_gene'
     *   tour-建议的连接顺序,长度为num_gene
     *
     * Returns a new join relation whose cheapest path is the best plan for
     * this join order.  NB: will return NULL if join order is invalid and
     * we can't modify it into a valid order.
     * 返回一个新的连接关系，其成本最低的路径是此连接顺序的最佳计划。
     * 如果join order无效，而且不能将其修改为有效的order，则返回NULL。
     *
     * The original implementation of this routine always joined in the specified
     * order, and so could only build left-sided plans (and right-sided and
     * mixtures, as a byproduct of the fact that make_join_rel() is symmetric).
     * It could never produce a "bushy" plan.  This had a couple of big problems,
     * of which the worst was that there are situations involving join order
     * restrictions where the only valid plans are bushy.
     * 这个处理过程的初始实现总是按照指定的顺序连接，因此只能构建左侧计划
     * (以及右侧和混合计划，这是make_join_rel()是对称的这一事实的副产品)。
     * 它永远不会产生一个“bushy”(N+M,其中N≥2,M≥2)的计划。
     * 这有几个大问题，其中最糟糕的是涉及到连接顺序限制的情况，其中唯一有效的计划是bushy的。
     *
     * The present implementation takes the given tour as a guideline, but
     * postpones joins that are illegal or seem unsuitable according to some
     * heuristic rules.  This allows correct bushy plans to be generated at need,
     * and as a nice side-effect it seems to materially improve the quality of the
     * generated plans.  Note however that since it's just a heuristic, it can
     * still fail in some cases.  (In particular, we might clump together
     * relations that actually mustn't be joined yet due to LATERAL restrictions;
     * since there's no provision for un-clumping, this must lead to failure.)
     * 目前的实施以给定的路线(tour)为指导，但根据一些启发式规则，延迟了非法或看似不合适的基因加入。
     * 这允许在需要时生成正确的bushy计划，这带来了额外的好处，似乎实质性地提高了生成计划的质量。
     * 但是请注意，由于它只是一个启发式的做法，在某些情况下它仍然可能失败。
     * (特别是，我们可能会将由于横向限制而实际上还不能被连接的关系组合在一起;由于没有关于非LATERAL的规定，这肯定会导致失败。)
     */
    RelOptInfo *
    gimme_tree(PlannerInfo *root, Gene *tour, int num_gene)
    {
        GeqoPrivateData *private = (GeqoPrivateData *) root->join_search_private;
        List       *clumps;
        int         rel_count;
    
        /*
         * Sometimes, a relation can't yet be joined to others due to heuristics
         * or actual semantic restrictions.  We maintain a list of "clumps" of
         * successfully joined relations, with larger clumps at the front. Each
         * new relation from the tour is added to the first clump it can be joined
         * to; if there is none then it becomes a new clump of its own. When we
         * enlarge an existing clump we check to see if it can now be merged with
         * any other clumps.  After the tour is all scanned, we forget about the
         * heuristics and try to forcibly join any remaining clumps.  If we are
         * unable to merge all the clumps into one, fail.
         * 有时，由于启发式或实际的语义限制，关系还不能连接到其他关系。
       * 因此保留了一个成功连接关系的“clumps”(聚类)链表，在此前有更大的clumps(聚类)。
       * 每个新关系从tour添加到第一个clump(聚类)，它可以加入;如果没有的话，它自己会构成一个clump(聚类)。
       * 当扩大现有的clump(聚类)时，需要检查它现在是否可以与其他clumps(聚类)合并。
       * 在所有的tour基因扫描之后，这时候不使用启发式规则，并试图强行加入任何剩余的clumps(聚类)中。
       * 如果我们不能把所有的聚类合并成一个种群，则失败。
         */
        clumps = NIL;
    
        for (rel_count = 0; rel_count < num_gene; rel_count++)//遍历基因即参与连接的关系
        {
            int         cur_rel_index;//当前索引
            RelOptInfo *cur_rel;//当前的关系
            Clump      *cur_clump;//当前的clump聚类
    
            /* 获取下一个输入的关系.Get the next input relation */
            cur_rel_index = (int) tour[rel_count];
            cur_rel = (RelOptInfo *) list_nth(private->initial_rels,
                                              cur_rel_index - 1);
    
            /* 放在一个单独的聚类clump中;Make it into a single-rel clump */
            cur_clump = (Clump *) palloc(sizeof(Clump));
            cur_clump->joinrel = cur_rel;
            cur_clump->size = 1;
    
            /* Merge it into the clumps list, using only desirable joins */
        //使用期望的连接方式(force=F)将它合并到clump(聚类)链表中
            clumps = merge_clump(root, clumps, cur_clump, num_gene, false);
        }
    
        if (list_length(clumps) > 1)//聚类链表>1
        {
            /* Force-join the remaining clumps in some legal order */
        //以传统的顺序加入到剩余的聚类中
            List       *fclumps;//链表
            ListCell   *lc;//元素
    
            fclumps = NIL;
            foreach(lc, clumps)
            {
                Clump      *clump = (Clump *) lfirst(lc);
          //(force=T)
                fclumps = merge_clump(root, fclumps, clump, num_gene, true);
            }
            clumps = fclumps;
        }
    
        /* Did we succeed in forming a single join relation? */
        if (list_length(clumps) != 1)//无法形成最终的结果关系,返回NULL
            return NULL;
    
        return ((Clump *) linitial(clumps))->joinrel;//成功,则返回结果Relation
    }
    
    
    //------------------------------ merge_clump
    /*
     * Merge a "clump" into the list of existing clumps for gimme_tree.
     * 将某个clump聚类合并到gimme_tree中生成的现存clumps聚类群中
     *
     * We try to merge the clump into some existing clump, and repeat if
     * successful.  When no more merging is possible, insert the clump
     * into the list, preserving the list ordering rule (namely, that
     * clumps of larger size appear earlier).
     * 尝试将clump合并到现有的clumps中，如果成功，则重复。
     * 当不再可能合并时，将clump插入到链表中，保留链表排序规则(即，更大的clump出现在前面)。
     * 
     * If force is true, merge anywhere a join is legal, even if it causes
     * a cartesian join to be performed.  When force is false, do only
     * "desirable" joins.
     * 如果force为true，则在连接合法的位置进行合并，即使这会导致执行笛卡尔连接。当力force为F时，只做“合适的”连接。
     */
    static List *
    merge_clump(PlannerInfo *root,//优化信息 
          List *clumps, //聚类链表
          Clump *new_clump, //新的聚类
          int num_gene,//基因格式
                bool force)//是否强制加入
    {
        ListCell   *prev;
        ListCell   *lc;
    
        /* Look for a clump that new_clump can join to */
      //验证新聚类能否加入
        prev = NULL;
        foreach(lc, clumps)//遍历链表
        {
            Clump      *old_clump = (Clump *) lfirst(lc);//原有的聚类
    
            if (force ||
                desirable_join(root, old_clump->joinrel, new_clump->joinrel))//如强制加入或者可按要求加入
            {
                RelOptInfo *joinrel;//
    
                /*
                 * Construct a RelOptInfo representing the join of these two input
                 * relations.  Note that we expect the joinrel not to exist in
                 * root->join_rel_list yet, and so the paths constructed for it
                 * will only include the ones we want.
           * 构造一个RelOptInfo，表示这两个输入关系的连接。
           * 注意，预期joinrel不会存在于root->join_rel_list中，因此为它构造的路径将只包含我们期望的路径。
                 */
                joinrel = make_join_rel(root,
                                        old_clump->joinrel,
                                        new_clump->joinrel);//构造连接新关系RelOptInfo
    
                /* 如连接顺序无效,继续搜索;Keep searching if join order is not valid */
                if (joinrel)
                {
                    /* Create paths for partitionwise joins. */
            //创建partitionwise连接
                    generate_partitionwise_join_paths(root, joinrel);
    
                    /*
                     * Except for the topmost scan/join rel, consider gathering
                     * partial paths.  We'll do the same for the topmost scan/join
                     * rel once we know the final targetlist (see
                     * grouping_planner).
             * 除了最上面的扫描/连接的关系，尝试gather partial(并行)访问路径。
             * 一旦我们知道最终的targetlist(参见grouping_planner)，将对最顶层的扫描/连接关系执行相同的操作。
                     */
                    if (old_clump->size + new_clump->size < num_gene)
                        generate_gather_paths(root, joinrel, false);
    
                    /* Find and save the cheapest paths for this joinrel */
            //设置成本最低的路径
                    set_cheapest(joinrel);
    
                    /* Absorb new clump into old */
            //把新的clump吸纳到旧的clump中,释放new_clump
                    old_clump->joinrel = joinrel;
                    old_clump->size += new_clump->size;
                    pfree(new_clump);
    
                    /* Remove old_clump from list */
            //从链表中删除old_clump
                    clumps = list_delete_cell(clumps, lc, prev);
    
                    /*
                     * Recursively try to merge the enlarged old_clump with
                     * others.  When no further merge is possible, we'll reinsert
                     * it into the list.
             * 递归地尝试将逐步扩大的old_clump与其他clump合并。
             * 当不能进一步合并时，我们将把它重新插入到链表中。
                     */
                    return merge_clump(root, clumps, old_clump, num_gene, force);
                }
            }
            prev = lc;
        }
    
        /*
         * No merging is possible, so add new_clump as an independent clump, in
         * proper order according to size.  We can be fast for the common case
         * where it has size 1 --- it should always go at the end.
       * 不可能合并，因此按照大小的适当顺序将new_clump添加为独立的clumps中。
       * 一般情况下，可以快速处理它的大小为1——总是在链表的最后。
         */
        if (clumps == NIL || new_clump->size == 1)
            return lappend(clumps, new_clump);//直接添加
    
        /* Check if it belongs at the front */
      //检查是否属于前面的clump
        lc = list_head(clumps);
        if (new_clump->size > ((Clump *) lfirst(lc))->size)
            return lcons(new_clump, clumps);
    
        /* Else search for the place to insert it */
      //搜索位置,插入之
        for (;;)
        {
            ListCell   *nxt = lnext(lc);
    
            if (nxt == NULL || new_clump->size > ((Clump *) lfirst(nxt))->size)
                break;              /* it belongs after 'lc', before 'nxt' */
            lc = nxt;
        }
        lappend_cell(clumps, lc, new_clump);
    
        return clumps;
    }
    

### 三、跟踪分析

测试表(13张表)和数据:

    
    
    drop table if exists t01;
    drop table if exists t02;
    drop table if exists t03;
    drop table if exists t04;
    drop table if exists t05;
    drop table if exists t06;
    drop table if exists t07;
    drop table if exists t08;
    drop table if exists t09;
    drop table if exists t10;
    drop table if exists t11;
    drop table if exists t12;
    drop table if exists t13;
    
    create table t01(c1 int,c2 varchar(20));
    create table t02(c1 int,c2 varchar(20));
    create table t03(c1 int,c2 varchar(20));
    create table t04(c1 int,c2 varchar(20));
    create table t05(c1 int,c2 varchar(20));
    create table t06(c1 int,c2 varchar(20));
    create table t07(c1 int,c2 varchar(20));
    create table t08(c1 int,c2 varchar(20));
    create table t09(c1 int,c2 varchar(20));
    create table t10(c1 int,c2 varchar(20));
    create table t11(c1 int,c2 varchar(20));
    create table t12(c1 int,c2 varchar(20));
    create table t13(c1 int,c2 varchar(20));
    
    insert into t01 select generate_series(1,100),'TEST'||generate_series(1,100);
    insert into t02 select generate_series(1,1000),'TEST'||generate_series(1,1000);
    insert into t03 select generate_series(1,10000),'TEST'||generate_series(1,10000);
    insert into t04 select generate_series(1,200),'TEST'||generate_series(1,200);
    insert into t05 select generate_series(1,4000),'TEST'||generate_series(1,4000);
    insert into t06 select generate_series(1,100000),'TEST'||generate_series(1,100000);
    insert into t07 select generate_series(1,100),'TEST'||generate_series(1,100);
    insert into t08 select generate_series(1,1000),'TEST'||generate_series(1,1000);
    insert into t09 select generate_series(1,10000),'TEST'||generate_series(1,10000);
    insert into t10 select generate_series(1,200),'TEST'||generate_series(1,200);
    insert into t11 select generate_series(1,4000),'TEST'||generate_series(1,4000);
    insert into t12 select generate_series(1,100000),'TEST'||generate_series(1,100000);
    insert into t13 select generate_series(1,100),'TEST'||generate_series(1,100);
    
    create index idx_t01_c1 on t01(c1);
    create index idx_t06_c1 on t06(c1);
    create index idx_t12_c1 on t12(c1);
    

测试SQL语句与执行计划如下:

    
    
    testdb=# explain verbose select * 
    from t01,t02,t03,t04,t05,t06,t07,t08,t09,t10,t11,t12,t13
    where t01.c1 = t02.c1 
    and t02.c1 = t03.c1
    and t03.c1 = t04.c1
    and t04.c1 = t05.c1
    and t05.c1 = t06.c1
    and t06.c1 = t07.c1
    and t07.c1 = t08.c1
    and t08.c1 = t09.c1
    and t09.c1 = t10.c1
    and t10.c1 = t11.c1
    and t11.c1 = t12.c1
    and t12.c1 = t13.c1;
                                               QUERY PLAN                                                                                        
                    
    -------------------------------------------------------------------------------------------------------------------------------------     Hash Join  (cost=404.93..597.44 rows=1 width=148)
       Output: t01.c1, t01.c2, t02.c1, t02.c2, t03.c1, t03.c2, t04.c1, t04.c2, t05.c1, t05.c2, t06.c1, t06.c2, t07.c1, t07.c2, t08.c1, t08.c2, t09.c1, t09.c2, t10.c1, t10.c2, t11.c1, t11.c2, t12.c1, t12.c2,
     t13.c1, t13.c2
       Hash Cond: (t03.c1 = t01.c1)
       ->  Seq Scan on public.t03  (cost=0.00..155.00 rows=10000 width=12)
             Output: t03.c1, t03.c2
       ->  Hash  (cost=404.92..404.92 rows=1 width=136)
             Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2, t11.c1, t11.c2, t08.c1, t08.c2, t05.c1, t05.c2, t06.c1, t06.c2, t01.c1, t
    01.c2
             ->  Nested Loop  (cost=327.82..404.92 rows=1 width=136)
                   Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2, t11.c1, t11.c2, t08.c1, t08.c2, t05.c1, t05.c2, t06.c1, t06.c2, t01
    .c1, t01.c2
                   Join Filter: (t02.c1 = t01.c1)
                   ->  Nested Loop  (cost=327.68..404.75 rows=1 width=126)
                         Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2, t11.c1, t11.c2, t08.c1, t08.c2, t05.c1, t05.c2, t06.c1, t06.c
    2
                         Join Filter: (t02.c1 = t06.c1)
                         ->  Hash Join  (cost=327.38..404.39 rows=1 width=113)
                               Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2, t11.c1, t11.c2, t08.c1, t08.c2, t05.c1, t05.c2
                               Hash Cond: (t05.c1 = t02.c1)
                               ->  Seq Scan on public.t05  (cost=0.00..62.00 rows=4000 width=12)
                                     Output: t05.c1, t05.c2
                               ->  Hash  (cost=327.37..327.37 rows=1 width=101)
                                     Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2, t11.c1, t11.c2, t08.c1, t08.c2
                                     ->  Hash Join  (cost=307.61..327.37 rows=1 width=101)
                                           Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2, t11.c1, t11.c2, t08.c1, t08.c2
                                           Hash Cond: (t08.c1 = t02.c1)
                                           ->  Seq Scan on public.t08  (cost=0.00..16.00 rows=1000 width=11)
                                                 Output: t08.c1, t08.c2
                                           ->  Hash  (cost=307.60..307.60 rows=1 width=90)
                                                 Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2, t11.c1, t11.c2
                                                 ->  Hash Join  (cost=230.59..307.60 rows=1 width=90)
                                                       Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2, t11.c1, t11.c2
                                                       Hash Cond: (t11.c1 = t02.c1)
                                                       ->  Seq Scan on public.t11  (cost=0.00..62.00 rows=4000 width=12)
                                                             Output: t11.c1, t11.c2
                                                       ->  Hash  (cost=230.58..230.58 rows=1 width=78)
                                                             Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2
                                                             ->  Nested Loop  (cost=37.71..230.58 rows=1 width=78)
                                                                   Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2, t12.c1, t12.c2
                                                                   Join Filter: (t02.c1 = t12.c1)
                                                                   ->  Hash Join  (cost=37.42..229.93 rows=1 width=65)
                                                                         Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2, t09.c1, t09.c2
                                                                         Hash Cond: (t09.c1 = t02.c1)
                                                                         ->  Seq Scan on public.t09  (cost=0.00..155.00 rows=10000 width=12)
                                                                               Output: t09.c1, t09.c2
                                                                         ->  Hash  (cost=37.41..37.41 rows=1 width=53)
                                                                               Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2
                                                                               ->  Hash Join  (cost=32.65..37.41 rows=1 width=53)
                                                                                     Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2, t04.c1, t04.c2
                                                                                     Hash Cond: (t04.c1 = t02.c1)
                                                                                     ->  Seq Scan on public.t04  (cost=0.00..4.00 rows=200 width=11)
                                                                                           Output: t04.c1, t04.c2
                                                                                     ->  Hash  (cost=32.62..32.62 rows=2 width=42)
                                                                                           Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2
                                                                                           ->  Hash Join  (cost=27.85..32.62 rows=2 width=42)
                                                                                                 Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2, t10.c1, t10.c2
                                                                                                 Hash Cond: (t10.c1 = t02.c1)
                                                                                                 ->  Seq Scan on public.t10  (cost=0.00..4.00 rows=200 width=11)
                                                                                                       Output: t10.c1, t10.c2
                                                                                                 ->  Hash  (cost=27.73..27.73 rows=10 width=31)
                                                                                                       Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2
                                                                                                       ->  Hash Join  (cost=6.50..27.73 rows=10 width=31)
                                                                                                             Output: t02.c1, t02.c2, t07.c1, t07.c2, t13.c1, t13.c2
                                                                                                             Hash Cond: (t02.c1 = t13.c1)
                                                                                                             ->  Hash Join  (cost=3.25..24.00 rows=100 width=21)
                                                                                                                   Output: t02.c1, t02.c2, t07.c1, t07.c2
                                                                                                                   Hash Cond: (t02.c1 = t07.c1)
                                                                                                                   ->  Seq Scan on public.t02  (cost=0.00..16.00 rows=1000 width=11)
                                                                                                                         Output: t02.c1, t02.c2
                                                                                                                   ->  Hash  (cost=2.00..2.00 rows=100 width=10)
                                                                                                                         Output: t07.c1, t07.c2
                                                                                                                         ->  Seq Scan on public.t07  (cost=0.00..2.00 rows=100 width=10)
                                                                                                                               Output: t07.c1, t07.c2
                                                                                                             ->  Hash  (cost=2.00..2.00 rows=100 width=10)
                                                                                                                   Output: t13.c1, t13.c2
                                                                                                                   ->  Seq Scan on public.t13  (cost=0.00..2.00 rows=100 width=10)
                                                                                                                         Output: t13.c1, t13.c2
                                                                   ->  Index Scan using idx_t12_c1 on public.t12  (cost=0.29..0.64 rows=1 width=13)
                                                                         Output: t12.c1, t12.c2
                                                                         Index Cond: (t12.c1 = t09.c1)
                         ->  Index Scan using idx_t06_c1 on public.t06  (cost=0.29..0.34 rows=1 width=13)
                               Output: t06.c1, t06.c2
                               Index Cond: (t06.c1 = t12.c1)
                   ->  Index Scan using idx_t01_c1 on public.t01  (cost=0.14..0.16 rows=1 width=10)
                         Output: t01.c1, t01.c2
                         Index Cond: (t01.c1 = t06.c1)
    (83 rows)
    
    testdb=# 
    

启动gdb,设置断点

    
    
    (gdb) b geqo
    Breakpoint 1 at 0x793ec6: file geqo_main.c, line 86.
    (gdb) c
    Continuing.
    
    Breakpoint 1, geqo (root=0x1fbf0f8, number_of_rels=13, initial_rels=0x1f0f698) at geqo_main.c:86
    86    int     edge_failures = 0;
    

输入参数:root为优化器信息,一共有13个参与连接的关系,initial_rels是13个参与连接关系的链表

    
    
    (gdb) p *initial_rels
    $1 = {type = T_List, length = 13, head = 0x1f0f670, tail = 0x1f0f888}
    

初始化遗传算法私有数据

    
    
    86    int     edge_failures = 0;
    (gdb) n
    97    root->join_search_private = (void *) &private;
    (gdb) 
    98    private.initial_rels = initial_rels;
    

设置种子值

    
    
    (gdb) n
    101   geqo_set_seed(root, Geqo_seed);
    

计算种群大小/迭代代数

    
    
    104   pool_size = gimme_pool_size(number_of_rels);
    (gdb) p Geqo_seed
    $2 = 0
    (gdb) n
    105   number_generations = gimme_number_generations(pool_size);
    (gdb) p pool_size
    $6 = 250
    (gdb) n
    111   pool = alloc_pool(root, pool_size, number_of_rels);
    (gdb) p number_generations
    $7 = 250
    

随机初始化种群,pool->data数组存储了组成该种群的染色体

    
    
    (gdb) n
    114   random_init_pool(root, pool);
    (gdb) n
    117   sort_pool(root, pool);    /* we have to do it only one time, since all
    (gdb) p *pool
    $20 = {data = 0x1f0f8d8, size = 250, string_length = 13}
    (gdb) p pool->data[0]->string
    $23 = (Gene *) 0x1f108f0
    (gdb) p *pool->data[0]->string
    $24 = 8
    (gdb) p pool->data[0].worth
    $50 = 635.99087618977478
    (gdb) p *pool->data[1]->string
    $25 = 7
    (gdb) p *pool->data[2]->string
    $26 = 6
    (gdb) p *pool->data[249].string
    $48 = 13
    (gdb) p pool->data[249].worth
    $49 = 601.3463999999999
    ...
    

开始进行迭代进化

    
    
    (gdb) n
    129   momma = alloc_chromo(root, pool->string_length);
    (gdb) 
    130   daddy = alloc_chromo(root, pool->string_length);
    (gdb) 
    137   edge_table = alloc_edge_table(root, pool->string_length);
    (gdb) 
    178   for (generation = 0; generation < number_generations; generation++)
    (gdb) p number_generations
    $52 = 250
    

利用线性偏差(bias)函数选择,然后交叉遗传

    
    
    181     geqo_selection(root, momma, daddy, pool, Geqo_selection_bias);
    (gdb) n
    185     gimme_edge_table(root, momma->string, daddy->string, pool->string_length, edge_table);
    (gdb) n
    187     kid = momma;
    (gdb) p *momma
    $1 = {string = 0x1f30460, worth = 637.36587618977478}
    (gdb) p *momma->string
    $2 = 11
    (gdb) p *daddy->string
    $3 = 8
    (gdb) p *daddy
    $4 = {string = 0x1f304e0, worth = 635.57048404744364}
    

遍历边界表,计算kid的成本,把kid放到种群中,进一步进化

    
    
    (gdb) 
    216     kid->worth = geqo_eval(root, kid->string, pool->string_length);
    (gdb) p *kid
    $5 = {string = 0x1f30460, worth = 637.36587618977478}
    (gdb) n
    219     spread_chromo(root, kid, pool);
    (gdb) p *kid
    $6 = {string = 0x1f30460, worth = 663.22435850797251}
    (gdb) n
    178   for (generation = 0; generation < number_generations; generation++)
    

下面考察进化过程中的geqo_eval函数,进入该函数,13个基因,tour为2

    
    
    (gdb) 
    216     kid->worth = geqo_eval(root, kid->string, pool->string_length);
    (gdb) step
    geqo_eval (root=0x1fbf0f8, tour=0x1f30460, num_gene=13) at geqo_eval.c:75
    75    mycontext = AllocSetContextCreate(CurrentMemoryContext,
    (gdb) p *tour
    $7 = 2
    

赋值/保存状态

    
    
    (gdb) n
    78    oldcxt = MemoryContextSwitchTo(mycontext);
    (gdb) 
    95    savelength = list_length(root->join_rel_list);
    (gdb) 
    96    savehash = root->join_rel_hash;
    (gdb) 
    97    Assert(root->join_rel_level == NULL);
    (gdb) 
    99    root->join_rel_hash = NULL;
    

进入geqo_eval->gimme_tree函数

    
    
    (gdb) 
    102   joinrel = gimme_tree(root, tour, num_gene);
    (gdb) step
    gimme_tree (root=0x1fbf0f8, tour=0x1f30460, num_gene=13) at geqo_eval.c:165
    165   GeqoPrivateData *private = (GeqoPrivateData *) root->join_search_private;
    

链表clumps初始化为NULL,开始循环,执行连接操作,tour数组保存了RTE的顺序

    
    
    (gdb) n
    180   clumps = NIL;
    (gdb) 
    182   for (rel_count = 0; rel_count < num_gene; rel_count++)
    (gdb) n
    189     cur_rel_index = (int) tour[rel_count];
    (gdb) p tour[0]
    $9 = 2
    (gdb) p tour[1]
    $10 = 12
    

循环添加到clumps中,直至所有的表都加入到clumps中或者无法产生有效的连接

    
    
    (gdb) n
    190     cur_rel = (RelOptInfo *) list_nth(private->initial_rels,
    (gdb) 
    194     cur_clump = (Clump *) palloc(sizeof(Clump));
    (gdb) 
    195     cur_clump->joinrel = cur_rel;
    (gdb) 
    196     cur_clump->size = 1;
    (gdb) 
    199     clumps = merge_clump(root, clumps, cur_clump, num_gene, false);
    (gdb) 
    (gdb) 
    182   for (rel_count = 0; rel_count < num_gene; rel_count++)
    

完成循环调用

    
    
    (gdb) b geqo_eval.c:218
    Breakpoint 2 at 0x793bf9: file geqo_eval.c, line 218.
    (gdb) c
    Continuing.
    
    Breakpoint 2, gimme_tree (root=0x1fbf0f8, tour=0x1f30460, num_gene=13) at geqo_eval.c:219
    219   if (list_length(clumps) != 1)
    

clumps链表中只有一个元素,该元素为13张表成功连接的访问路径

    
    
    (gdb) p *clumps
    $11 = {type = T_List, length = 1, head = 0x1ea2e20, tail = 0x1ea2e20}
    $12 = {joinrel = 0x1ee4ef8, size = 13}
    (gdb) p *((Clump *)clumps->head->data.ptr_value)->joinrel
    $13 = {type = T_RelOptInfo, reloptkind = RELOPT_JOINREL, relids = 0x1ee34b0, rows = 100, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x1ee5110, pathlist = 0x1ee5a78, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x1ee5ee0, cheapest_total_path = 0x1ee5ee0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x1ee5fa0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 0, 
      reltablespace = 0, rtekind = RTE_JOIN, min_attr = 0, max_attr = 0, attr_needed = 0x0, attr_widths = 0x0, 
      lateral_vars = 0x0, lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 0, tuples = 0, allvisfrac = 0, 
      subroot = 0x0, subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, 
      fdwroutine = 0x0, fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, 
      baserestrictcost = {startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, 
      has_eclass_joins = false, consider_partitionwise_join = false, top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, 
      boundinfo = 0x0, partition_qual = 0x0, part_rels = 0x0, partexprs = 0x0, nullable_partexprs = 0x0, 
      partitioned_child_rels = 0x0}
    

geqo_eval->gimme_tree函数返回

    
    
    (gdb) n
    222   return ((Clump *) linitial(clumps))->joinrel;
    

回到geqo_eval函数,设置适应度,还原现场等

    
    
    (gdb) 
    113     Path     *best_path = joinrel->cheapest_total_path;
    (gdb) n
    115     fitness = best_path->total_cost;
    (gdb) 
    124   root->join_rel_list = list_truncate(root->join_rel_list,
    (gdb) 
    126   root->join_rel_hash = savehash;
    (gdb) 
    129   MemoryContextSwitchTo(oldcxt);
    (gdb) 
    130   MemoryContextDelete(mycontext);
    (gdb) 
    132   return fitness;
    (gdb) p fitness
    $14 = 680.71399161308523
    

回到函数geqo,继续迭代

    
    
    geqo (root=0x1fbf0f8, number_of_rels=13, initial_rels=0x1f0f698) at geqo_main.c:219
    219     spread_chromo(root, kid, pool);
    (gdb) 
    178   for (generation = 0; generation < number_generations; generation++)
    

完成循环迭代

    
    
    (gdb) b geqo_main.c:229
    Breakpoint 3 at 0x79407a: file geqo_main.c, line 229.
    (gdb) c
    Continuing.
    Breakpoint 3, geqo (root=0x1fbf0f8, number_of_rels=13, initial_rels=0x1f0f698) at geqo_main.c:260
    260   best_tour = (Gene *) pool->data[0].string;
    

最佳的访问节点路径(存储在best_tour数组中)

    
    
    (gdb) p best_tour[0]
    $17 = 2
    (gdb) p best_tour[1]
    $18 = 7
    (gdb) p best_tour[12]
    $19 = 3
    (gdb) p best_tour[13]
    

最佳的最终结果关系

    
    
    (gdb) p *best_rel
    $21 = {type = T_RelOptInfo, reloptkind = RELOPT_JOINREL, relids = 0x1f3d098, rows = 1, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x1f3d7e0, pathlist = 0x1f3e148, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x1f3e550, cheapest_total_path = 0x1f3e550, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x1f3e670, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 0, 
      reltablespace = 0, rtekind = RTE_JOIN, min_attr = 0, max_attr = 0, attr_needed = 0x0, attr_widths = 0x0, 
      lateral_vars = 0x0, lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 0, tuples = 0, allvisfrac = 0, 
      subroot = 0x0, subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, 
      fdwroutine = 0x0, fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, 
      baserestrictcost = {startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, 
      has_eclass_joins = false, consider_partitionwise_join = false, top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, 
      boundinfo = 0x0, partition_qual = 0x0, part_rels = 0x0, partexprs = 0x0, nullable_partexprs = 0x0, 
      partitioned_child_rels = 0x0}
    

清理现场,并返回

    
    
    (gdb) n
    274   free_chromo(root, daddy);
    (gdb) 
    277   free_edge_table(root, edge_table);
    (gdb) 
    294   free_pool(root, pool);
    (gdb) 
    297   root->join_search_private = NULL;
    (gdb) 
    299   return best_rel;
    (gdb) 
    300 }
    

DONE!

### 四、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)  
[十分钟搞懂遗传算法](https://www.imooc.com/article/23911)  
[你和遗传算法的距离也许只差这一文](https://cloud.tencent.com/developer/article/1103422)  
[智能优化方法及其应用-遗传算法](http://www.icst.pku.edu.cn/F/zLian/course/IOMA/4-GA1.pdf)

