先前的章节介绍了query_planner中主计划函数make_one_rel的实现逻辑，再继续介绍之前有必要Review
PG的Optimizer的机制，理论结合代码实现以便更好的理解代码。PG的Optimizer机制在源代码中的README文件(src/backend/optimizer/README)有相关说明。

### 一、Paths and Join Pairs

Paths and Join Pairs  
访问路径和连接对

> During the planning/optimizing process, we build "Path" trees representing  
>  the different ways of doing a query. We select the cheapest Path that  
>  generates the desired relation and turn it into a Plan to pass to the  
>  executor. (There is pretty nearly a one-to-one correspondence between the  
>  Path and Plan trees, but Path nodes omit info that won't be needed during  
>  planning, and include info needed for planning that won't be needed by the  
>  executor.)

在计划/优化过程中,通过构建"Path"树表示执行查询的不同方法,在这些方法中,选择生成所需Relation的最低成本路径(Path结构体)并转换为Plan结构体传递给执行器.(Path和Plan树两者之间几乎存在一对一的对应关系,但Path节点省略了在计划期间不需要但包含了在执行期间不需要而在计划期间需要的信息)

> The optimizer builds a RelOptInfo structure for each base relation used in  
>  the query. Base rels are either primitive tables, or subquery subselects  
>  that are planned via a separate recursive invocation of the planner. A  
>  RelOptInfo is also built for each join relation that is considered during  
>  planning. A join rel is simply a combination of base rels. There is only  
>  one join RelOptInfo for any given set of baserels --- for example, the join  
>  {A B C} is represented by the same RelOptInfo no matter whether we build it  
>  by joining A and B first and then adding C, or joining B and C first and  
>  then adding A, etc. These different means of building the joinrel are
represented as Paths. For each RelOptInfo we build a list of Paths that  
>  represent plausible ways to implement the scan or join of that relation.  
>  Once we've considered all the plausible Paths for a rel, we select the one  
>  that is cheapest according to the planner's cost estimates. The final plan  
>  is derived from the cheapest Path for the RelOptInfo that includes all the  
>  base rels of the query.

优化器为查询中使用到的每一个基础关系(base relation)创建RelOptInfo结构体.基础关系(base
relation)包括原始表(primitive
table),子查询(通过独立的计划器递归调用生成计划)以及参与连接的Relation.连接生成的关系(join
rel)是基础关系的组合(可以理解为通过连接运算生成的新关系).对于给定的基础关系集合,只有一个连接的RelOptInfo结构体生成,而这个RelOptInfo如何生成(比如基础关系集合{A
B C},A和B可以先连接然后与C连接,当然也可以B和C先连接然后在与A连接)并不关心.这些构建join
rel的方法通过Paths表示.对每一个RelOptInfo,优化器会构建Paths链表来表示这些"貌似有理的"实现扫描或者连接的方法.对于这些所有可能的路径,优化器会根据成本估算选择成本最低的访问路径.最后的执行计划从RelOptInfo(最终生成的新关系)成本最低的访问路径产生.

> Possible Paths for a primitive table relation include plain old sequential  
>  scan, plus index scans for any indexes that exist on the table, plus bitmap  
>  index scans using one or more indexes. Specialized RTE types, such as  
>  function RTEs, may have only one possible Path.

访问原始表的可能路径包括普通的顺序扫描/基于索引的索引扫描/位图索引扫描.对于某些特殊的RTE类型,比如函数RTEs,可能只有一种可能的路径.

> Joins always occur using two RelOptInfos. One is outer, the other inner.  
>  Outers drive lookups of values in the inner. In a nested loop, lookups of  
>  values in the inner occur by scanning the inner path once per outer tuple  
>  to find each matching inner row. In a mergejoin, inner and outer rows are  
>  ordered, and are accessed in order, so only one scan is required to perform  
>  the entire join: both inner and outer paths are scanned in-sync. (There's  
>  not a lot of difference between inner and outer in a mergejoin...) In a  
>  hashjoin, the inner is scanned first and all its rows are entered in a  
>  hashtable, then the outer is scanned and for each row we lookup the join  
>  key in the hashtable.

连接通常出现在两个RelOptInfo之间,俗称外部表和内部表,其中外部表又称驱动表.在Nested
Loop连接方式,对于外部的每一个元组,都会访问内部表以扫描满足条件的数据行.Merge
Join连接方式,外部表和内部表的元组已排序,顺序访问外部表和内部表的每一个元组即可,这种方式只需要同步扫描一次.Hash
Join连接方式,首先会扫描内部表并建立HashTable,然后扫描外部表,对于外部表的每一行扫描哈希表找出匹配行.

> A Path for a join relation is actually a tree structure, with the topmost  
>  Path node representing the last-applied join method. It has left and right  
>  subpaths that represent the scan or join methods used for the two input  
>  relations.

join
rel的访问路径实际上是一种树状结构,最顶层的路径节点表示最好应用的连接方法.这颗树有左右两个子路径(subpath)用以表示两个relations的扫描或连接方法.

### 二、Join Tree Construction

Join Tree Construction  
连接树的构造

> The optimizer generates optimal query plans by doing a more-or-less  
>  exhaustive search through the ways of executing the query. The best Path  
>  tree is found by a recursive process:

优化器尽可能的通过穷尽搜索的方法生成最优的查询执行计划,最优的访问路径树通过以下递归过程实现:

> 1).Take each base relation in the query, and make a RelOptInfo structure  
>  for it. Find each potentially useful way of accessing the relation,  
>  including sequential and index scans, and make Paths representing those  
>  ways. All the Paths made for a given relation are placed in its  
>  RelOptInfo.pathlist. (Actually, we discard Paths that are obviously  
>  inferior alternatives before they ever get into the pathlist --- what  
>  ends up in the pathlist is the cheapest way of generating each potentially  
>  useful sort ordering and parameterization of the relation.) Also create a  
>  RelOptInfo.joininfo list including all the join clauses that involve this  
>  relation. For example, the WHERE clause "tab1.col1 = tab2.col1" generates  
>  entries in both tab1 and tab2's joininfo lists.

1).为查询中每个基础关系生成RelOptInfo结构体.为每个基础关系生成顺序扫描或索引扫描访问路径.这些生成的访问路径存储在RelOptInfo.pathlist链表中(实际上,在此过程中优化器已经抛弃了明显不合理的访问路径,在pathlist中的路径是生成排序路径和参数化Relation的最可能路径).在此期间,会生成RelOptInfo.joininfo链表,用于保存与此Relation相关的索引的连接语句(join
clauses).比如,WHERE语句"tab1.col1 =
tab2.col1"在tab1和tab2的joininfo链表中会产生相应的数据结构(entries).

> If we have only a single base relation in the query, we are done.  
>  Otherwise we have to figure out how to join the base relations into a  
>  single join relation.

如果查询中只有一个基础关系,优化器已完成所有工作,否则的话,优化器需要得出如何连接基础关系,从而得到一个新关系(通过join连接而来).

> 2).Normally, any explicit JOIN clauses are "flattened" so that we just  
>  have a list of relations to join. However, FULL OUTER JOIN clauses are  
>  never flattened, and other kinds of JOIN might not be either, if the  
>  flattening process is stopped by join_collapse_limit or from_collapse_limit  
>  restrictions. Therefore, we end up with a planning problem that contains  
>  lists of relations to be joined in any order, where any individual item  
>  might be a sub-list that has to be joined together before we can consider  
>  joining it to its siblings. We process these sub-problems recursively,  
>  bottom up. Note that the join list structure constrains the possible join  
>  orders, but it doesn't constrain the join implementation method at each  
>  join (nestloop, merge, hash), nor does it say which rel is considered outer  
>  or inner at each join. We consider all these possibilities in building  
>  Paths. We generate a Path for each feasible join method, and select the  
>  cheapest Path.

2)通常来说,显式的JOIN语句已被扁平化(flattened)处理,优化器可以直接根据关系链表进行连接.但是,全外连接(FULL OUTER
JOIN)以及某些类型的JOIN无法扁平化(比如由于join_collapse_limit或from_collapse_limit这些约束条件).这里会遇到这么一个问题:尝试以任意顺序进行连接的关系链表,链表中的子链表必须在两两连接之前进行连接.优化器会同自底向上的方式递归处理这些子问题.注意:连接链表限制了连接顺序,但没有限制连接的实现方法或者那个关系是内表或外表,这些问题在生成访问路径时解决.优化器会为每个可行的连接方式生成访问路径,并选择其中成本最低的那个.

> For each planning problem, therefore, we will have a list of relations  
>  that are either base rels or joinrels constructed per sub-join-lists.  
>  We can join these rels together in any order the planner sees fit.  
>  The standard (non-GEQO) planner does this as follows:

对于每一个计划过程中出现的问题,优化器把每一个构成子连接链表(sub-join-list)的  
基础关系或连接关系存储在一个链表中.优化器会根据看起来合适的顺序连接这些关系.标准的(非遗传算法,即动态规划算法)计划器执行以下操作:

> Consider joining each RelOptInfo to each other RelOptInfo for which there  
>  is a usable joinclause, and generate a Path for each possible join method  
>  for each such pair. (If we have a RelOptInfo with no join clauses, we have  
>  no choice but to generate a clauseless Cartesian-product join; so we  
>  consider joining that rel to each other available rel. But in the presence  
>  of join clauses we will only consider joins that use available join  
>  clauses. Note that join-order restrictions induced by outer joins and  
>  IN/EXISTS clauses are also checked, to ensure that we find a workable join  
>  order in cases where those restrictions force a clauseless join to be
done.)

对于每一个可用的连接条件,考虑使用两两连接的方式,为每一对RelOptInfo的连接生成访问路径.如果没有连接条件,那么会使用笛卡尔连接,因此会优先考虑连接其他可用的Relation.

> If we only had two relations in the list, we are done: we just pick  
>  the cheapest path for the join RelOptInfo. If we had more than two, we now  
>  need to consider ways of joining join RelOptInfos to each other to make  
>  join RelOptInfos that represent more than two list items.

如果链表中只有2个Relations,优化器已完成所有工作,为参与连接的RelOptInfo挑选最优的访问路径即可.否则的话(>3个Relations),优化器需要考虑如何两两进行连接.

> The join tree is constructed using a "dynamic programming" algorithm:  
>  in the first pass (already described) we consider ways to create join rels  
>  representing exactly two list items. The second pass considers ways  
>  to make join rels that represent exactly three list items; the next pass,  
>  four items, etc. The last pass considers how to make the final join  
>  relation that includes all list items --- obviously there can be only one  
>  join rel at this top level, whereas there can be more than one join rel  
>  at lower levels. At each level we use joins that follow available join  
>  clauses, if possible, just as described for the first level.

连接树通过"动态规划"算法构造:  
在第一轮(先前已描述)中,优化器已完成两个Relations的连接方式;第二轮,优化器考虑如何创建三个Relations的join
rels;下一轮,四个Relations,以此类推.最后一轮,考虑如何构造包含所有Relations的join rel.显然,在最高层,只有一个join
rel,而在低层则可能会有多个join rel.在每一个层次上,如前所述,如果可以的话,优化器会按照可用的连接条件进行连接.

> For example:  
>  SELECT *  
>  FROM tab1, tab2, tab3, tab4  
>  WHERE tab1.col = tab2.col AND  
>  tab2.col = tab3.col AND  
>  tab3.col = tab4.col

>

> Tables 1, 2, 3, and 4 are joined as:  
>  {1 2},{2 3},{3 4}  
>  {1 2 3},{2 3 4}  
>  {1 2 3 4}  
>  (other possibilities will be excluded for lack of join clauses)

>

> SELECT *  
>  FROM tab1, tab2, tab3, tab4  
>  WHERE tab1.col = tab2.col AND  
>  tab1.col = tab3.col AND  
>  tab1.col = tab4.col

>

> Tables 1, 2, 3, and 4 are joined as:  
>  {1 2},{1 3},{1 4}  
>  {1 2 3},{1 3 4},{1 2 4}  
>  {1 2 3 4}

> We consider left-handed plans (the outer rel of an upper join is a joinrel,  
>  but the inner is always a single list item); right-handed plans (outer rel  
>  is always a single item); and bushy plans (both inner and outer can be  
>  joins themselves). For example, when building {1 2 3 4} we consider  
>  joining {1 2 3} to {4} (left-handed), {4} to {1 2 3} (right-handed), and  
>  {1 2} to {3 4} (bushy), among other choices. Although the jointree  
>  scanning code produces these potential join combinations one at a time,  
>  all the ways to produce the same set of joined base rels will share the  
>  same RelOptInfo, so the paths produced from different join combinations  
>  that produce equivalent joinrels will compete in add_path().

下面来看看left-handed计划,bushy计划和right-handed计划.比如,在构建{1 2 3
4}4个关系的连接时,在众多的选择中存在以下方式,left-handed:{1 2 3} 连接 {4},right-handed:{4}连接{1 2
3}和bushy:{1 2}连接{3 4}.虽然扫描连接树时一次产生这些潜在的连接组合,但是所有产生相同连接base
rels集合的方法会共享相同的RelOptInfo的数据结构,因此这些不同的连接组合在生成等价的join rel时会调用add_path方法时相互PK.

> The dynamic-programming approach has an important property that's not  
>  immediately obvious: we will finish constructing all paths for a given  
>  relation before we construct any paths for relations containing that rel.  
>  This means that we can reliably identify the "cheapest path" for each rel  
>  before higher-level relations need to know that. Also, we can safely  
>  discard a path when we find that another path for the same rel is better,  
>  without worrying that maybe there is already a reference to that path in  
>  some higher-level join path. Without this, memory management for paths  
>  would be much more complicated.

动态规划方法有一个重要的特性(自底向上):那就是在构建高层RelOptInfo的访问路径前,下层的RelOptInfo的访问路径已明确,而且优化器确保该访问路径是成本最低的.

> Once we have built the final join rel, we use either the cheapest path  
>  for it or the cheapest path with the desired ordering (if that's cheaper  
>  than applying a sort to the cheapest other path).

一旦完成了最终结果join rel的构建,存在两条路径:成本最低或者按排序的要求最低

> If the query contains one-sided outer joins (LEFT or RIGHT joins), or  
>  IN or EXISTS WHERE clauses that were converted to semijoins or antijoins,  
>  then some of the possible join orders may be illegal. These are excluded  
>  by having join_is_legal consult a side list of such "special" joins to see  
>  whether a proposed join is illegal. (The same consultation allows it to  
>  see which join style should be applied for a valid join, ie, JOIN_INNER,  
>  JOIN_LEFT, etc.)

如果查询语句存在外连接或者转换为半连接或反连接的IN或EXISTS语句,那么有些可能的连接顺序是非法的.优化器通过额外的方法进行处理.

### 三、参考资料

[README](https://github.com/postgres/postgres/blob/master/src/backend/optimizer/README)

