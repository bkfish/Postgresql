这一小节是Review
PG的Optimizer机制的第二篇，同样的，PG的Optimizer机制在源代码中的README文件(src/backend/optimizer/README)有相关说明，这一小节介绍了优化函数的全流程和相关的数据结构等。

### 一、Optimizer Functions

Optimizer Functions-查询优化函数

> The primary entry point is planner().  
>  planner() //主入口  
>  set up for recursive handling of subqueries  
>  -subquery_planner()//planner->subquery_planner  
>  pull up sublinks and subqueries from rangetable, if possible  
>  canonicalize qual  
>  Attempt to simplify WHERE clause to the most useful form; this includes  
>  flattening nested AND/ORs and detecting clauses that are duplicated in  
>  different branches of an OR.  
>  simplify constant expressions  
>  process sublinks  
>  convert Vars of outer query levels into Params  
>  \--grouping_planner()//planner->subquery_planner->grouping_planner  
>  preprocess target list for non-SELECT queries  
>  handle UNION/INTERSECT/EXCEPT, GROUP BY, HAVING, aggregates,  
>  ORDER BY, DISTINCT, LIMIT  
>  \---query_planner()//subquery_planner->grouping_planner->query_planner  
>  make list of base relations used in query  
>  split up the qual into restrictions (a=1) and joins (b=c)  
>  find qual clauses that enable merge and hash joins  
>  \----make_one_rel()//...grouping_planner->query_planner->make_one_rel  
>  set_base_rel_pathlists() //为每一个RelOptInfo生成访问路径  
>  find seqscan and all index paths for each base relation  
>  find selectivity of columns used in joins  
>  make_rel_from_joinlist() //使用遗传算法或动态规划算法构造连接路径  
>  hand off join subproblems to a plugin, GEQO, or standard_join_search()  
>  \-----standard_join_search()//这是动态规划算法  
>  call join_search_one_level() for each level of join tree needed  
>  join_search_one_level():  
>  For each joinrel of the prior level, do make_rels_by_clause_joins()  
>  if it has join clauses, or make_rels_by_clauseless_joins() if not.  
>  Also generate "bushy plan" joins between joinrels of lower levels.  
>  Back at standard_join_search(), generate gather paths if needed for  
>  each newly constructed joinrel, then apply set_cheapest() to extract  
>  the cheapest path for it.  
>  Loop back if this wasn't the top join level.  
>  Back at grouping_planner:  
>  do grouping (GROUP BY) and aggregation//在最高层处理分组/聚集/唯一过滤/排序/控制输出元组数目等  
>  do window functions  
>  make unique (DISTINCT)  
>  do sorting (ORDER BY)  
>  do limit (LIMIT/OFFSET)  
>  Back at planner():  
>  convert finished Path tree into a Plan tree  
>  do final cleanup after planning

### 二、Optimizer Data Structures

Optimizer Data Structures  
数据结构

> PlannerGlobal - global information for a single planner invocation  
>  PlannerInfo - information for planning a particular Query (we make  
>  a separate PlannerInfo node for each sub-Query)  
>  RelOptInfo - a relation or joined relations  
>  RestrictInfo - WHERE clauses, like "x = 3" or "y = z"  
>  (note the same structure is used for restriction and  
>  join clauses)  
>  Path - every way to generate a RelOptInfo(sequential,index,joins)  
>  SeqScan - represents a sequential scan plan //顺序扫描  
>  IndexPath - index scan //索引扫描  
>  BitmapHeapPath - top of a bitmapped index scan //位图索引扫描  
>  TidPath - scan by CTID //CTID扫描  
>  SubqueryScanPath - scan a subquery-in-FROM //FROM子句中的子查询扫描  
>  ForeignPath - scan a foreign table, foreign join or foreign upper-relation
//FDW  
>  CustomPath - for custom scan providers //定制化扫描  
>  AppendPath - append multiple subpaths together //多个子路径APPEND,常见于集合操作  
>  MergeAppendPath - merge multiple subpaths, preserving their common sort
order //保持顺序的APPEND  
>  ResultPath - a childless Result plan node (used for FROM-less
SELECT)//结果路径(如SELECT 2+2)  
>  MaterialPath - a Material plan node //物化路径  
>  UniquePath - remove duplicate rows (either by hashing or sorting) //去除重复行路径  
>  GatherPath - collect the results of parallel workers //并行  
>  GatherMergePath - collect parallel results, preserving their common sort
order //并行,保持顺序  
>  ProjectionPath - a Result plan node with child (used for projection) //投影  
>  ProjectSetPath - a ProjectSet plan node applied to some sub-path
//投影(应用于子路径上)  
>  SortPath - a Sort plan node applied to some sub-path //排序  
>  GroupPath - a Group plan node applied to some sub-path //分组  
>  UpperUniquePath - a Unique plan node applied to some sub-path
//应用于子路径的Unique Plan  
>  AggPath - an Agg plan node applied to some sub-path //应用于子路径的聚集  
>  GroupingSetsPath - an Agg plan node used to implement GROUPING SETS //分组集合  
>  MinMaxAggPath - a Result plan node with subplans performing MIN/MAX //最大最小  
>  WindowAggPath - a WindowAgg plan node applied to some sub-path
//应用于子路径的窗口函数  
>  SetOpPath - a SetOp plan node applied to some sub-path //应用于子路径的集合操作  
>  RecursiveUnionPath - a RecursiveUnion plan node applied to two sub-paths
//递归UNION  
>  LockRowsPath - a LockRows plan node applied to some sub-path
//应用于子路径的的LockRows  
>  ModifyTablePath - a ModifyTable plan node applied to some sub-path(s)
//应用于子路径的数据表更新(如INSERT/UPDATE操作等)  
>  LimitPath - a Limit plan node applied to some sub-path//应用于子路径的LIMIT  
>  NestPath - nested-loop joins//嵌套循环连接  
>  MergePath - merge joins//Merge Join  
>  HashPath - hash joins//Hash Join  
>  EquivalenceClass - a data structure representing a set of values known
equal  
>  PathKey - a data structure representing the sort ordering of a path

> The optimizer spends a good deal of its time worrying about the ordering  
>  of the tuples returned by a path. The reason this is useful is that by  
>  knowing the sort ordering of a path, we may be able to use that path as  
>  the left or right input of a mergejoin and avoid an explicit sort step.  
>  Nestloops and hash joins don't really care what the order of their inputs  
>  is, but mergejoin needs suitably ordered inputs. Therefore, all paths  
>  generated during the optimization process are marked with their sort order  
>  (to the extent that it is known) for possible use by a higher-level merge.

优化器在元组的排序上面花费了不少时间,原因是为了在Merge Join时避免专门的排序步骤.

> It is also possible to avoid an explicit sort step to implement a user's  
>  ORDER BY clause if the final path has the right ordering already, so the  
>  sort ordering is of interest even at the top level. grouping_planner() will  
>  look for the cheapest path with a sort order matching the desired order,  
>  then compare its cost to the cost of using the cheapest-overall path and  
>  doing an explicit sort on that.  
>  When we are generating paths for a particular RelOptInfo, we discard a path  
>  if it is more expensive than another known path that has the same or better  
>  sort order. We will never discard a path that is the only known way to  
>  achieve a given sort order (without an explicit sort, that is). In this  
>  way, the next level up will have the maximum freedom to build mergejoins  
>  without sorting, since it can pick from any of the paths retained for its  
>  inputs.

以上解释了优化器为什么要返回一个未排序和排好序的两个路径给上层的原因.排好序的路径可以直接用于Merge Join或者排序.

### 三、参考资料

[README](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/README)

