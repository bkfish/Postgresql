本节简单介绍了PG执行查询语句中优化器部分（Optimizer）的相关函数和数据结构总体说明。查询优化包括查询逻辑优化和查询物理优化，查询逻辑优化是指使用关系代数中的等价规则，通过选择下推、投影下推、连接交换等方法对SQL语句进行优化;查询物理优化是指通过CBO对各种物理访问数据的方法进行评估，得出最优的执行计划。

### 一、总体说明

下面是PG源码目录(/src/backend/optimizer)中的README文件对优化器相关函数和数据结构的总体说明:

    
    
    Optimizer Functions
    -------------------    
    The primary entry point is planner().
    
    planner()//优化器主入口函数
    set up for recursive handling of subqueries//为子查询配置处理器(递归方式)
    -subquery_planner()//调用(子)查询优化函数
     pull up sublinks and subqueries from rangetable, if possible//可以的话,上拉子链接和子查询
     canonicalize qual//表达式规范化
         Attempt to simplify WHERE clause to the most useful form; this includes
         flattening nested AND/ORs and detecting clauses that are duplicated in
         different branches of an OR.//简化WHERE语句
     simplify constant expressions//简化常量表达式
     process sublinks//处理子链接
     convert Vars of outer query levels into Params//转换外查询的Vars变量到Params中
    --grouping_planner()//
      preprocess target list for non-SELECT queries//预处理非SELECT语句的投影列
      handle UNION/INTERSECT/EXCEPT, GROUP BY, HAVING, aggregates,//处理集合操作/聚集函数/排序等
        ORDER BY, DISTINCT, LIMIT
    --query_planner()//
       make list of base relations used in query//构造查询中的基表链表
       split up the qual into restrictions (a=1) and joins (b=c)//拆分表达式为限制条件和连接
       find qual clauses that enable merge and hash joins//查找可以让Merge和Hash连接生效的表达式
    ----make_one_rel()//
         set_base_rel_pathlists()//设置基表路径链表
          find seqscan and all index paths for each base relation//遍历每个基表,寻找顺序扫描和所有可能的索引扫描路径
          find selectivity of columns used in joins//查找连接中使用的列的选择性
         make_rel_from_joinlist()//通过join链表构造Relation
          hand off join subproblems to a plugin, GEQO, or standard_join_search()//
    -----standard_join_search()//标准的连接搜索函数
          call join_search_one_level() for each level of join tree needed//每一个join tree调用join_search_one_level
          join_search_one_level():
            For each joinrel of the prior level, do make_rels_by_clause_joins()//对于上一层的每一个joinrel,执行make_rels_by_clause_joins
            if it has join clauses, or make_rels_by_clauseless_joins() if not.
            Also generate "bushy plan" joins between joinrels of lower levels.
          Back at standard_join_search(), generate gather paths if needed for//回到standard_join_search函数,需要的话,收集相关的路径并应用set_cheapest函数获取代价最小的路径
          each newly constructed joinrel, then apply set_cheapest() to extract
          the cheapest path for it.
          Loop back if this was not the top join level.//如果不是最顶层连接,循环
      Back at grouping_planner://回到grouping_planner函数
      do grouping (GROUP BY) and aggregation//处理分组和聚集
      do window functions//处理窗口函数
      make unique (DISTINCT)//处理唯一性
      do sorting (ORDER BY)//处理排序
      do limit (LIMIT/OFFSET)//处理Limit
    Back at planner()://回到planner函数
    convert finished Path tree into a Plan tree//转换最终的路径树到计划树
    do final cleanup after planning//收尾工作
    
    
    Optimizer Data Structures
    -------------------------    
    PlannerGlobal   - global information for a single planner invocation//全局优化信息
    
    PlannerInfo     - information for planning a particular Query (we make//某个Planner的优化信息
                      a separate PlannerInfo node for each sub-Query)
    
    RelOptInfo      - a relation or joined relations//某个Relation(包括连接)的优化信息
    
     RestrictInfo   - WHERE clauses, like "x = 3" or "y = z"//限制条件
                      (note the same structure is used for restriction and
                       join clauses)
    
     Path           - every way to generate a RelOptInfo(sequential,index,joins)//构造该关系(注意:中间结果也是关系的一种)的路径
      SeqScan       - represents a sequential scan plan
      IndexPath     - index scan
      BitmapHeapPath - top of a bitmapped index scan
      TidPath       - scan by CTID
      SubqueryScanPath - scan a subquery-in-FROM
      ForeignPath   - scan a foreign table, foreign join or foreign upper-relation
      CustomPath    - for custom scan providers
      AppendPath    - append multiple subpaths together
      MergeAppendPath - merge multiple subpaths, preserving their common sort order
      ResultPath    - a childless Result plan node (used for FROM-less SELECT)
      MaterialPath  - a Material plan node
      UniquePath    - remove duplicate rows (either by hashing or sorting)
      GatherPath    - collect the results of parallel workers
      GatherMergePath - collect parallel results, preserving their common sort order
      ProjectionPath - a Result plan node with child (used for projection)
      ProjectSetPath - a ProjectSet plan node applied to some sub-path
      SortPath      - a Sort plan node applied to some sub-path
      GroupPath     - a Group plan node applied to some sub-path
      UpperUniquePath - a Unique plan node applied to some sub-path
      AggPath       - an Agg plan node applied to some sub-path
      GroupingSetsPath - an Agg plan node used to implement GROUPING SETS
      MinMaxAggPath - a Result plan node with subplans performing MIN/MAX
      WindowAggPath - a WindowAgg plan node applied to some sub-path
      SetOpPath     - a SetOp plan node applied to some sub-path
      RecursiveUnionPath - a RecursiveUnion plan node applied to two sub-paths
      LockRowsPath  - a LockRows plan node applied to some sub-path
      ModifyTablePath - a ModifyTable plan node applied to some sub-path(s)
      LimitPath     - a Limit plan node applied to some sub-path
      NestPath      - nested-loop joins
      MergePath     - merge joins
      HashPath      - hash joins
    
     EquivalenceClass - a data structure representing a set of values known equal//等价类
    
     PathKey        - a data structure representing the sort ordering of a path//排序键
    

下一节开始将根据总体说明中的函数逐个进行分析解读.

### 二、小结

1、优化器函数总览：大体介绍了优化器函数的调用过程等信息；  
2、数据结构：优化器相关的数据结构，如PlannerInfo等。

