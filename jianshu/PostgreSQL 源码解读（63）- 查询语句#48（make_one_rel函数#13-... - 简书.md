本节大体介绍了make_one_rel中的make_rel_from_joinlist->standard_join_search函数的实现逻辑，该函数是PG使用动态规划算法构造连接路径的实现。

### 一、源码解读

上节已解读了make_rel_from_joinlist->standard_join_search函数的主实现逻辑,下面重点介绍该函数中的join_search_one_level函数.

    
    
     /*
      * join_search_one_level
      *    Consider ways to produce join relations containing exactly 'level'
      *    jointree items.  (This is one step of the dynamic-programming method
      *    embodied in standard_join_search.)  Join rel nodes for each feasible
      *    combination of lower-level rels are created and returned in a list.
      *    Implementation paths are created for each such joinrel, too.
      *    规划如何生成包含匹配Leve(比如2个关系的连接/3个关系的连接等)连接关系。
      *    (这是在standard_join_search中体现的动态规划算法的一个步骤。)
      *    为较低Leve的关系创建新的连接关系亦即访问路径,通过链表的方式返回(root->join_rel_level)。 
      *
      * level: level of rels we want to make this time
      * root->join_rel_level[j], 1 <= j < level, is a list of rels containing j items
      * level:关系的level,比如是2个关系还是3个关系的连接
      *
      * The result is returned in root->join_rel_level[level].
      * 结果通过root->join_rel_level[level]
      */
     void
     join_search_one_level(PlannerInfo *root, int level)
     {
         List      **joinrels = root->join_rel_level;
         ListCell   *r;
         int         k;
     
         Assert(joinrels[level] == NIL);
     
         /* Set join_cur_level so that new joinrels are added to proper list */
         root->join_cur_level = level;//当前的Level
     
         /*
          * First, consider left-sided and right-sided plans, in which rels of
          * exactly level-1 member relations are joined against initial relations.
          * We prefer to join using join clauses, but if we find a rel of level-1
          * members that has no join clauses, we will generate Cartesian-product
          * joins against all initial rels not already contained in it.
          * 首先,规划left-sided和right-sided的计划,这些计划已由初始关系连接为level-1级的Relation.
          * PG使用连接条件进行连接,但如果发现level-1成员中没有连接条件,那么PG将会
          * 为未包含此条件的初始关系生成笛卡尔积.
          */
         foreach(r, joinrels[level - 1])//遍历上一级生成的关系
         {
             RelOptInfo *old_rel = (RelOptInfo *) lfirst(r);//获取上一级的RelOptInfo
     
             if (old_rel->joininfo != NIL || old_rel->has_eclass_joins ||
                 has_join_restriction(root, old_rel))//存在连接条件
             {
                 /*
                  * There are join clauses or join order restrictions relevant to
                  * this rel, so consider joins between this rel and (only) those
                  * initial rels it is linked to by a clause or restriction.
                  * 存在与此rel相关的连接条件或连接顺序限制，
                  * 因此仅规划此rel与通过条件子句或约束条件链接在一起的初始rels.
                  *
                  * At level 2 this condition is symmetric, so there is no need to
                  * look at initial rels before this one in the list; we already
                  * considered such joins when we were at the earlier rel.  (The
                  * mirror-image joins are handled automatically by make_join_rel.)
                  * In later passes (level > 2), we join rels of the previous level
                  * to each initial rel they don't already include but have a join
                  * clause or restriction with.
                  * leve=2时，这个条件是对称的，所以不需要在关注链表中此rel前的rels;
                  * 在处理在此rel前的rels时,已处理这样的连接.(make_join_rel函数自动处理镜像连接)。
                  * level>2时，PG将上一级别生成的rels逐一与尚未处理的初始rel(存在连接条件或约束条件)进行连接.
                  *
                  */
                 ListCell   *other_rels;
     
                 if (level == 2)     /* consider remaining initial rels */
                     other_rels = lnext(r);//level = 2,只需关注此rel之后的rel
                 else                /* consider all initial rels */
                     other_rels = list_head(joinrels[1]);//level > 2,从第1级开始尝试
     
                 make_rels_by_clause_joins(root,
                                           old_rel,
                                           other_rels);//创建连接
             }
             else//不存在连接条件
             {
                 /*
                  * Oops, we have a relation that is not joined to any other
                  * relation, either directly or by join-order restrictions.
                  * Cartesian product time.
                  * 有一个relation与其他relation没有连接条件(直接或通过join-order约束)
                  * 笛卡尔时间到了! 
                  *
                  * We consider a cartesian product with each not-already-included
                  * initial rel, whether it has other join clauses or not.  At
                  * level 2, if there are two or more clauseless initial rels, we
                  * will redundantly consider joining them in both directions; but
                  * such cases aren't common enough to justify adding complexity to
                  * avoid the duplicated effort.
                  * 考察每一个尚未处理的初始rel(无论其是否有约束条件).
                  * 在level 2,如存在2个或以上的无条件初始rels,PG可能会出现重复处理的情况.
                  */
                 make_rels_by_clauseless_joins(root,
                                               old_rel,
                                               list_head(joinrels[1]));//创建无条件连接
             }
         }
     
         /*
          * Now, consider "bushy plans" in which relations of k initial rels are
          * joined to relations of level-k initial rels, for 2 <= k <= level-2.
          * 现在考察"稠密计划",其中k level的rels与level - k的rel想连接.其中:2 <= k <= level-2
          *
          * We only consider bushy-plan joins for pairs of rels where there is a
          * suitable join clause (or join order restriction), in order to avoid
          * unreasonable growth of planning time.
          * 这里只考虑存在连接条件(或者join-order限制)的关系对,以避免计划时间的大幅增加
          */
         for (k = 2;; k++)
         {
             int         other_level = level - k;
     
             /*
              * Since make_join_rel(x, y) handles both x,y and y,x cases, we only
              * need to go as far as the halfway point.
              */
             if (k > other_level)
                 break;
     
             foreach(r, joinrels[k])
             {
                 RelOptInfo *old_rel = (RelOptInfo *) lfirst(r);
                 ListCell   *other_rels;
                 ListCell   *r2;
     
                 /*
                  * We can ignore relations without join clauses here, unless they
                  * participate in join-order restrictions --- then we might have
                  * to force a bushy join plan.
                  */
                 if (old_rel->joininfo == NIL && !old_rel->has_eclass_joins &&
                     !has_join_restriction(root, old_rel))
                     continue;
     
                 if (k == other_level)
                     other_rels = lnext(r);  /*同一层次,只考虑余下的rel,only consider remaining rels */
                 else
                     other_rels = list_head(joinrels[other_level]);//不同层次,尝试所有的
     
                 for_each_cell(r2, other_rels)
                 {
                     RelOptInfo *new_rel = (RelOptInfo *) lfirst(r2);
     
                     if (!bms_overlap(old_rel->relids, new_rel->relids))//relids不存在包含关系
                     {
                         /*
                          * OK, we can build a rel of the right level from this
                          * pair of rels.  Do so if there is at least one relevant
                          * join clause or join order restriction.
                          */
                         if (have_relevant_joinclause(root, old_rel, new_rel) ||
                             have_join_order_restriction(root, old_rel, new_rel))//存在连接条件或者join-order约束
                         {
                             (void) make_join_rel(root, old_rel, new_rel);//创建连接
                         }
                     }
                 }
             }
         }
     
         /*----------          * Last-ditch effort: if we failed to find any usable joins so far, force
          * a set of cartesian-product joins to be generated.  This handles the
          * special case where all the available rels have join clauses but we
          * cannot use any of those clauses yet.  This can only happen when we are
          * considering a join sub-problem (a sub-joinlist) and all the rels in the
          * sub-problem have only join clauses with rels outside the sub-problem.
          * An example is
          *
          *      SELECT ... FROM a INNER JOIN b ON TRUE, c, d, ...
          *      WHERE a.w = c.x and b.y = d.z;
          *
          * If the "a INNER JOIN b" sub-problem does not get flattened into the
          * upper level, we must be willing to make a cartesian join of a and b;
          * but the code above will not have done so, because it thought that both
          * a and b have joinclauses.  We consider only left-sided and right-sided
          * cartesian joins in this case (no bushy).
          *----------          */
         if (joinrels[level] == NIL)
         {
             /*
              * This loop is just like the first one, except we always call
              * make_rels_by_clauseless_joins().
              */
             foreach(r, joinrels[level - 1])
             {
                 RelOptInfo *old_rel = (RelOptInfo *) lfirst(r);
     
                 make_rels_by_clauseless_joins(root,
                                               old_rel,
                                               list_head(joinrels[1]));
             }
     
             /*----------              * When special joins are involved, there may be no legal way
              * to make an N-way join for some values of N.  For example consider
              *
              * SELECT ... FROM t1 WHERE
              *   x IN (SELECT ... FROM t2,t3 WHERE ...) AND
              *   y IN (SELECT ... FROM t4,t5 WHERE ...)
              *
              * We will flatten this query to a 5-way join problem, but there are
              * no 4-way joins that join_is_legal() will consider legal.  We have
              * to accept failure at level 4 and go on to discover a workable
              * bushy plan at level 5.
              *
              * However, if there are no special joins and no lateral references
              * then join_is_legal() should never fail, and so the following sanity
              * check is useful.
              *----------              */
             if (joinrels[level] == NIL &&
                 root->join_info_list == NIL &&
                 !root->hasLateralRTEs)
                 elog(ERROR, "failed to build any %d-way joins", level);
         }
     }
     
    
    //------------------------------------------------------------------- has_join_restriction
     /*
      * has_join_restriction
      *      Detect whether the specified relation has join-order restrictions,
      *      due to being inside an outer join or an IN (sub-SELECT),
      *      or participating in any LATERAL references or multi-rel PHVs.
      *      判断传入的relation是否含有join-order限制条件.存在于外连接/IN(sub-SELECT)子查询/LATERAL依赖/多关系PHVs
      *
      * Essentially, this tests whether have_join_order_restriction() could
      * succeed with this rel and some other one.  It's OK if we sometimes
      * say "true" incorrectly.  (Therefore, we don't bother with the relatively
      * expensive has_legal_joinclause test.)
      */
     static bool
     has_join_restriction(PlannerInfo *root, RelOptInfo *rel)
     {
         ListCell   *l;
     
         if (rel->lateral_relids != NULL || rel->lateral_referencers != NULL)
             return true;//存在lateral
     
         foreach(l, root->placeholder_list)
         {
             PlaceHolderInfo *phinfo = (PlaceHolderInfo *) lfirst(l);
     
             if (bms_is_subset(rel->relids, phinfo->ph_eval_at) &&
                 !bms_equal(rel->relids, phinfo->ph_eval_at))
                 return true;//PHVs
         }
     
         foreach(l, root->join_info_list)
         {
             SpecialJoinInfo *sjinfo = (SpecialJoinInfo *) lfirst(l);
     
             /* ignore full joins --- other mechanisms preserve their ordering */
             if (sjinfo->jointype == JOIN_FULL)
                 continue;//不考虑全外连接
     
             /* ignore if SJ is already contained in rel */
             if (bms_is_subset(sjinfo->min_lefthand, rel->relids) &&
                 bms_is_subset(sjinfo->min_righthand, rel->relids))
                 continue;//SJ在rel中,不考虑
     
             /* restricted if it overlaps LHS or RHS, but doesn't contain SJ */
             if (bms_overlap(sjinfo->min_lefthand, rel->relids) ||
                 bms_overlap(sjinfo->min_righthand, rel->relids))
                 return true;
         }
     
         return false;
     }
     
    
    //------------------------------------------------------------------- make_rels_by_clause_joins
     /*
      * make_rels_by_clause_joins
      *    Build joins between the given relation 'old_rel' and other relations
      *    that participate in join clauses that 'old_rel' also participates in
      *    (or participate in join-order restrictions with it).
      *    The join rels are returned in root->join_rel_level[join_cur_level].
      *   创建old_rel和其他rel的连接(两者存在连接条件)
      *
      * Note: at levels above 2 we will generate the same joined relation in
      * multiple ways --- for example (a join b) join c is the same RelOptInfo as
      * (b join c) join a, though the second case will add a different set of Paths
      * to it.  This is the reason for using the join_rel_level mechanism, which
      * automatically ensures that each new joinrel is only added to the list once.
      * 注意:在level > 2时,PG会通过多种方式生成同样的连接rel(joined relation).
      * 比如:(a join b) join c与(b join c) join a最终结果是一样的RelOptInfo,虽然第
      * 2种方法会添加一些不同的访问路径集合在其中.
      * 这其实是使用join_rel_level的原因,确保每个新joinrel只加入到合适的链表中
      *
      * 'old_rel' is the relation entry for the relation to be joined
      * 'other_rels': the first cell in a linked list containing the other
      * rels to be considered for joining
      * old-rel:需要连接的rel
      * other-rel:候选关系链表中的的第一个cell
      *
      * Currently, this is only used with initial rels in other_rels, but it
      * will work for joining to joinrels too.
      * 看起来似乎只对other_rels中的初始rels有用,但其实对于连接生成的joinrels同样会生效.
      */
     static void
     make_rels_by_clause_joins(PlannerInfo *root,
                               RelOptInfo *old_rel,
                               ListCell *other_rels)
     {
         ListCell   *l;
     
         for_each_cell(l, other_rels)//遍历链表
         {
             RelOptInfo *other_rel = (RelOptInfo *) lfirst(l);//获取其中的RelOptInfo
     
             if (!bms_overlap(old_rel->relids, other_rel->relids) &&
                 (have_relevant_joinclause(root, old_rel, other_rel) ||
                  have_join_order_restriction(root, old_rel, other_rel)))//reldis不同而且存在连接关系&连接顺序约束
             {
                 (void) make_join_rel(root, old_rel, other_rel);//创建连接
             }
         }
     }
    
    //---------------------------------------------------- have_relevant_joinclause
     /*
      * have_relevant_joinclause
      *      Detect whether there is a joinclause that involves
      *      the two given relations.
      *      给定两个relations,检查两者是否存在连接条件
      *
      * Note: the joinclause does not have to be evaluable with only these two
      * relations.  This is intentional.  For example consider
      *      SELECT * FROM a, b, c WHERE a.x = (b.y + c.z)
      * If a is much larger than the other tables, it may be worthwhile to
      * cross-join b and c and then use an inner indexscan on a.x.  Therefore
      * we should consider this joinclause as reason to join b to c, even though
      * it can't be applied at that join step.
      * 注意:连接条件不一定是等值连接,
      *      比如:SELECT * FROM a, b, c WHERE a.x = (b.y + c.z),只要a.x大于b.y + c.z即可
      */
     bool
     have_relevant_joinclause(PlannerInfo *root,
                              RelOptInfo *rel1, RelOptInfo *rel2)
     {
         bool        result = false;
         List       *joininfo;
         Relids      other_relids;
         ListCell   *l;
     
         /*
          * We could scan either relation's joininfo list; may as well use the
          * shorter one.
          * 获取relation中joininfo链表较少的那个
          */
         if (list_length(rel1->joininfo) <= list_length(rel2->joininfo))
         {
             joininfo = rel1->joininfo;
             other_relids = rel2->relids;
         }
         else
         {
             joininfo = rel2->joininfo;
             other_relids = rel1->relids;
         }
     
         foreach(l, joininfo)//遍历
         {
             RestrictInfo *rinfo = (RestrictInfo *) lfirst(l);
     
             if (bms_overlap(other_relids, rinfo->required_relids))//存在交集
             {
                 result = true;//存在连接条件
                 break;
             }
         }
     
         /*
          * We also need to check the EquivalenceClass data structure, which might
          * contain relationships not emitted into the joininfo lists.
          * 检查等价类
          */
         if (!result && rel1->has_eclass_joins && rel2->has_eclass_joins)
             result = have_relevant_eclass_joinclause(root, rel1, rel2);//存在等价类连接条件
     
         return result;
     }
     
    
    //---------------------------------------------------- have_join_order_restriction
     /*
      * have_join_order_restriction
      *      Detect whether the two relations should be joined to satisfy
      *      a join-order restriction arising from special or lateral joins.
      *      检查两个relations是否需要连接以满足join-order限制(由于special/lateral连接引起)
      *
      * In practice this is always used with have_relevant_joinclause(), and so
      * could be merged with that function, but it seems clearer to separate the
      * two concerns.  We need this test because there are degenerate cases where
      * a clauseless join must be performed to satisfy join-order restrictions.
      * Also, if one rel has a lateral reference to the other, or both are needed
      * to compute some PHV, we should consider joining them even if the join would
      * be clauseless.
      * 在实践中，这通常与have_relevance _join子()一起使用，因此可以与该函数合并，
      * 但分离这两个关注点似乎更为清晰。在一些退化的情况下需要这个测试，
      * 必须执行无语法连接以满足连接顺序限制。
      * 另外，如果一个rel与另一个rel有一个lateral引用，
      * 或者两者都需要计算一些PHV，那么我们应该考虑加入它们，即使连接是无连接条件的。
      * 
      * Note: this is only a problem if one side of a degenerate outer join
      * contains multiple rels, or a clauseless join is required within an
      * IN/EXISTS RHS; else we will find a join path via the "last ditch" case in
      * join_search_one_level().  We could dispense with this test if we were
      * willing to try bushy plans in the "last ditch" case, but that seems much
      * less efficient.
      * 注意:只有当简并外部连接的一侧包含多个rels时，
      *      或者在IN/EXISTS RHS中需要一个无修饰的连接时，才会出现这个问题;
      * 否则，将通过join_search_one_level()中的“last ditch”
      * 找到连接路径。如果愿意在“稠密计划”的情况下进行大量的尝试，
      * 那么可以省去这个测试，但这似乎效率要低得多。
      */
     bool
     have_join_order_restriction(PlannerInfo *root,
                                 RelOptInfo *rel1, RelOptInfo *rel2)
     {
         bool        result = false;
         ListCell   *l;
     
         /*
          * If either side has a direct lateral reference to the other, attempt the
          * join regardless of outer-join considerations.
          */
         if (bms_overlap(rel1->relids, rel2->direct_lateral_relids) ||
             bms_overlap(rel2->relids, rel1->direct_lateral_relids))
             return true;//relids与lateral relids存在交集,返回T
     
         /*
          * Likewise, if both rels are needed to compute some PlaceHolderVar,
          * attempt the join regardless of outer-join considerations.  (This is not
          * very desirable, because a PHV with a large eval_at set will cause a lot
          * of probably-useless joins to be considered, but failing to do this can
          * cause us to fail to construct a plan at all.)
          */
         foreach(l, root->placeholder_list)//遍历PHV
         {
             PlaceHolderInfo *phinfo = (PlaceHolderInfo *) lfirst(l);
     
             if (bms_is_subset(rel1->relids, phinfo->ph_eval_at) &&
                 bms_is_subset(rel2->relids, phinfo->ph_eval_at))
                 return true;
         }
     
         /*
          * It's possible that the rels correspond to the left and right sides of a
          * degenerate outer join, that is, one with no joinclause mentioning the
          * non-nullable side; in which case we should force the join to occur.
          *
          * Also, the two rels could represent a clauseless join that has to be
          * completed to build up the LHS or RHS of an outer join.
          */
         foreach(l, root->join_info_list)//遍历连接链表
         {
             SpecialJoinInfo *sjinfo = (SpecialJoinInfo *) lfirst(l);
     
             /* ignore full joins --- other mechanisms handle them */
             if (sjinfo->jointype == JOIN_FULL)
                 continue;
     
             /* Can we perform the SJ with these rels? */
             if (bms_is_subset(sjinfo->min_lefthand, rel1->relids) &&
                 bms_is_subset(sjinfo->min_righthand, rel2->relids))
             {
                 result = true;
                 break;
             }
             if (bms_is_subset(sjinfo->min_lefthand, rel2->relids) &&
                 bms_is_subset(sjinfo->min_righthand, rel1->relids))
             {
                 result = true;
                 break;
             }
     
             /*
              * Might we need to join these rels to complete the RHS?  We have to
              * use "overlap" tests since either rel might include a lower SJ that
              * has been proven to commute with this one.
              */
             if (bms_overlap(sjinfo->min_righthand, rel1->relids) &&
                 bms_overlap(sjinfo->min_righthand, rel2->relids))
             {
                 result = true;
                 break;
             }
     
             /* Likewise for the LHS. */
             if (bms_overlap(sjinfo->min_lefthand, rel1->relids) &&
                 bms_overlap(sjinfo->min_lefthand, rel2->relids))
             {
                 result = true;
                 break;
             }
         }
     
         /*
          * We do not force the join to occur if either input rel can legally be
          * joined to anything else using joinclauses.  This essentially means that
          * clauseless bushy joins are put off as long as possible. The reason is
          * that when there is a join order restriction high up in the join tree
          * (that is, with many rels inside the LHS or RHS), we would otherwise
          * expend lots of effort considering very stupid join combinations within
          * its LHS or RHS.
          */
         if (result)
         {
             if (has_legal_joinclause(root, rel1) ||
                 has_legal_joinclause(root, rel2))
                 result = false;
         }
     
         return result;
     }
     
    
    

### 二、跟踪分析

创建测试数据表并生成测试数据:

    
    
    drop table if exists a;
    drop table if exists b;
    drop table if exists c;
    drop table if exists d;
    drop table if exists e;
    drop table if exists f;
    
    create table a(c1 int,c2 varchar(20));
    create table b(c1 int,c2 varchar(20));
    create table c(c1 int,c2 varchar(20));
    create table d(c1 int,c2 varchar(20));
    create table e(c1 int,c2 varchar(20));
    create table f(c1 int,c2 varchar(20));
    
    insert into a select generate_series(1,100),'TEST'||generate_series(1,100);
    insert into b select generate_series(1,1000),'TEST'||generate_series(1,1000);
    insert into c select generate_series(1,10000),'TEST'||generate_series(1,10000);
    insert into d select generate_series(1,200),'TEST'||generate_series(1,200);
    insert into e select generate_series(1,4000),'TEST'||generate_series(1,4000);
    insert into f select generate_series(1,100000),'TEST'||generate_series(1,100000);
    

测试脚本:

    
    
    testdb=# explain verbose select a.*,b.c1,c.c2,d.c2,e.c1,f.c2
    from a inner join b on a.c1=b.c1,c,d,e inner join f on e.c1 = f.c1 and e.c1 < 100
    where a.c1=f.c1 and b.c1=c.c1 and c.c1 = d.c1 and d.c1 = e.c1;
                                                    QUERY PLAN                                                
    ----------------------------------------------------------------------------------------------------------     Nested Loop  (cost=101.17..2218.24 rows=2 width=42)
       Output: a.c1, a.c2, b.c1, c.c2, d.c2, e.c1, f.c2
       Join Filter: (a.c1 = b.c1)
       ->  Hash Join  (cost=3.25..196.75 rows=100 width=22)
             Output: a.c1, a.c2, c.c2, c.c1
             Hash Cond: (c.c1 = a.c1)
             ->  Seq Scan on public.c  (cost=0.00..155.00 rows=10000 width=12)
                   Output: c.c1, c.c2
             ->  Hash  (cost=2.00..2.00 rows=100 width=10)
                   Output: a.c1, a.c2
                   ->  Seq Scan on public.a  (cost=0.00..2.00 rows=100 width=10)
                         Output: a.c1, a.c2
       ->  Materialize  (cost=97.92..2014.00 rows=5 width=32)
             Output: b.c1, d.c2, d.c1, e.c1, f.c2, f.c1
             ->  Hash Join  (cost=97.92..2013.97 rows=5 width=32)
                   Output: b.c1, d.c2, d.c1, e.c1, f.c2, f.c1
                   Hash Cond: (f.c1 = b.c1)
                   ->  Seq Scan on public.f  (cost=0.00..1541.00 rows=100000 width=13)
                         Output: f.c1, f.c2
                   ->  Hash  (cost=97.86..97.86 rows=5 width=19)
                         Output: b.c1, d.c2, d.c1, e.c1
                         ->  Hash Join  (cost=78.10..97.86 rows=5 width=19)
                               Output: b.c1, d.c2, d.c1, e.c1
                               Hash Cond: (b.c1 = e.c1)
                               ->  Seq Scan on public.b  (cost=0.00..16.00 rows=1000 width=4)
                                     Output: b.c1, b.c2
                               ->  Hash  (cost=78.04..78.04 rows=5 width=15)
                                     Output: d.c2, d.c1, e.c1
                                     ->  Hash Join  (cost=73.24..78.04 rows=5 width=15)
                                           Output: d.c2, d.c1, e.c1
                                           Hash Cond: (d.c1 = e.c1)
                                           ->  Seq Scan on public.d  (cost=0.00..4.00 rows=200 width=11)
                                                 Output: d.c1, d.c2
                                           ->  Hash  (cost=72.00..72.00 rows=99 width=4)
                                                 Output: e.c1
                                                 ->  Seq Scan on public.e  (cost=0.00..72.00 rows=99 width=4)
                                                       Output: e.c1
                                                       Filter: (e.c1 < 100)
    (38 rows)
    

测试SQL语句的连接关系:a-b,a-f,b-c,c-d,d-e,e-f  
注:根据先前章节的知识,该SQL语句存在等价类{a.c1 b.c1 c.c1 d.c1 e.c1 f.c1}

启动gdb跟踪

    
    
    (gdb) b join_search_one_level
    Breakpoint 1 at 0x755667: file joinrels.c, line 67.
    (gdb) c
    Continuing.
    
    Breakpoint 1, join_search_one_level (root=0x3006e28, level=2) at joinrels.c:67
    67      List      **joinrels = root->join_rel_level;
    

查看优化器信息(root)

    
    
    (gdb) p *root
    $13 = {type = T_PlannerInfo, parse = 0x2fa3410, glob = 0x3008578, query_level = 1, parent_root = 0x0, plan_params = 0x0, 
      outer_params = 0x0, simple_rel_array = 0x2f510e8, simple_rel_array_size = 9, simple_rte_array = 0x2f51178, 
      all_baserels = 0x2f53dd8, nullable_baserels = 0x0, join_rel_list = 0x2fcb5c8, join_rel_hash = 0x0, 
      join_rel_level = 0x2fcafe8, join_cur_level = 2, init_plans = 0x0, cte_plan_ids = 0x0, multiexpr_params = 0x0, 
      eq_classes = 0x2f52cb8, canon_pathkeys = 0x2fcb718, left_join_clauses = 0x0, right_join_clauses = 0x0, 
      full_join_clauses = 0x0, join_info_list = 0x0, append_rel_list = 0x0, rowMarks = 0x0, placeholder_list = 0x0, 
      fkey_list = 0x0, query_pathkeys = 0x0, group_pathkeys = 0x0, window_pathkeys = 0x0, distinct_pathkeys = 0x0, 
      sort_pathkeys = 0x0, part_schemes = 0x0, initial_rels = 0x2fcaf18, upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, 
      upper_targets = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, processed_tlist = 0x2f4f718, grouping_map = 0x0, minmax_aggs = 0x0, 
      planner_cxt = 0x2e87040, total_table_pages = 627, tuple_fraction = 0, limit_tuples = -1, qual_security_level = 0, 
      inhTargetKind = INHKIND_NONE, hasJoinRTEs = true, hasLateralRTEs = false, hasDeletedRTEs = false, hasHavingQual = false, 
      hasPseudoConstantQuals = false, hasRecursion = false, wt_param_id = -1, non_recursive_path = 0x0, curOuterRels = 0x0, 
      curOuterParams = 0x0, join_search_private = 0x0, partColsUpdated = false}
    

root->simple_rel_array_size=9,数组中有9个元素,从1-8(下标为0的元素无用)分别是1->RTE_RELATION/16775,2->RTE_RELATION/16778,3->RTE_JOIN,4->RTE_RELATION/16781,5->RTE_RELATION/16784,6->RTE_RELATION/16787,7->RTE_RELATION/16790,8->RTE_JOIN

    
    
      oid  | relname 
    -------+---------     16775 | a          -->1
     16778 | b          -->2    
     16781 | c          -->4
     16784 | d          -->5
     16787 | e          -->6
     16790 | f          -->7
    (6 rows)
    

进入join_search_one_level函数,level=2,开始循环遍历joinrels

    
    
    (gdb) n
    74      root->join_cur_level = level;
    (gdb) 
    83      foreach(r, joinrels[level - 1])
    (gdb) n
    85          RelOptInfo *old_rel = (RelOptInfo *) lfirst(r);
    (gdb) 
    87          if (old_rel->joininfo != NIL || old_rel->has_eclass_joins ||
    (gdb) 
    105             if (level == 2)     /* consider remaining initial rels */
    (gdb) 
    106                 other_rels = lnext(r);
    (gdb) 
    110             make_rels_by_clause_joins(root,
    

[level=2]进入make_rels_by_clause_joins函数

    
    
    (gdb) step
    make_rels_by_clause_joins (root=0x3006e28, old_rel=0x3008258, other_rels=0x2fcaf48) at joinrels.c:280
    280     for_each_cell(l, other_rels)
    

[level=2]由于存在等价类{a.c1 b.c1 c.c1 d.c1 e.c1
f.c1},因此这一步骤会两两连接构造新的关系,ab,ac,ad,ae,af,bc,bd,...

    
    
    (gdb) n
    282         RelOptInfo *other_rel = (RelOptInfo *) lfirst(l);
    (gdb) 
    284         if (!bms_overlap(old_rel->relids, other_rel->relids) &&
    (gdb) 
    285             (have_relevant_joinclause(root, old_rel, other_rel) ||
    (gdb) 
    284         if (!bms_overlap(old_rel->relids, other_rel->relids) &&
    (gdb) 
    288             (void) make_join_rel(root, old_rel, other_rel);
    (gdb) n
    280     for_each_cell(l, other_rels)
    

[level=2]调用make_join_rel函数后,查看root->join_rel_level[2],relids=6=2+4,这是1号(关系a)和2号(关系b)RTE的连接.

    
    
    (gdb) p *root->join_rel_level[2]
    $6 = {type = T_List, length = 1, head = 0x2fcb5f8, tail = 0x2fcb5f8}
    (gdb) p *(Node *)root->join_rel_level[2]->head->data.ptr_value
    $7 = {type = T_RelOptInfo}
    (gdb) p *(RelOptInfo *)root->join_rel_level[2]->head->data.ptr_value
    $8 = {type = T_RelOptInfo, reloptkind = RELOPT_JOINREL, relids = 0x2fcb050, rows = 100, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x2fcb068, pathlist = 0x2fcba08, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x0, cheapest_total_path = 0x0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 0, reltablespace = 0, 
      rtekind = RTE_JOIN, min_attr = 0, max_attr = 0, attr_needed = 0x0, attr_widths = 0x0, lateral_vars = 0x0, 
      lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 0, tuples = 0, allvisfrac = 0, subroot = 0x0, 
      subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, fdwroutine = 0x0, 
      fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, baserestrictcost = {
        startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, has_eclass_joins = true, 
      top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, partition_qual = 0x0, part_rels = 0x0, 
      partexprs = 0x0, nullable_partexprs = 0x0, partitioned_child_rels = 0x0}
    (gdb) set $tmp=(RelOptInfo *)root->join_rel_level[2]->head->data.ptr_value
    (gdb) p *$tmp->relids->words
    $10 = 6
    

[level=2]继续循环,下几组分别是ac,ad,ae,af

    
    
    (gdb) p *$tmp->relids->words
    $12 = 18/34/66/130
    

[level=2]完成对关系a的两两连接

    
    
    (gdb) n
    291 }
    (gdb) 
    join_search_one_level (root=0x3006e28, level=2) at joinrels.c:89
    89          {
    (gdb) n
    83      foreach(r, joinrels[level - 1])
    

[level=2]类似的,处理b/c/d/e/f,两两形成连接,一共有15种组合(6!/(2!*(6-2)!))

    
    
    (gdb) 
    83      foreach(r, joinrels[level - 1])
    (gdb) 
    142     for (k = 2;; k++)
    (gdb) p *root->join_rel_level[2]
    $44 = {type = T_List, length = 15, head = 0x2fcb5f8, tail = 0x2fd7f78}
    

[level=2]完成level=2的调用,level2的relids组合有1&2,1&4,1&5,1&6,1&7,2&4,2&5,2&6,2&7,4&5,4&6,4&7,5&6,5&7,6&7

    
    
    (gdb) 
    standard_join_search (root=0x3006e28, levels_needed=6, initial_rels=0x2fcaf18) at allpaths.c:2757
    2757            foreach(lc, root->join_rel_level[lev])
    

开始level=3的调用

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 1, join_search_one_level (root=0x3006e28, level=3) at joinrels.c:67
    67      List      **joinrels = root->join_rel_level;
    

[level=3]遍历level=2的RelOptInfo(两两连接形成的新关系)

    
    
    (gdb) 
    83      foreach(r, joinrels[level - 1])
    

[level=3]与level=2不同,选择初始的RelOptInfo进行连接,而不是同级的rels

    
    
    ...
    (gdb) 
    108                 other_rels = list_head(joinrels[1]);
    

[level=3]完成第一轮的循环,root->join_rel_level[3]链表中有4个Node(RelOptInfo),其relids分别是22/38/70/134,即1&2&4,1&2&5,1&2&6,1&2&7

    
    
    (gdb) p *((RelOptInfo *)root->join_rel_level[3]->head->data.ptr_value)->relids->words
    $55 = 22
    (gdb) p *((RelOptInfo *)root->join_rel_level[3]->head->next->data.ptr_value)->relids->words
    $56 = 38
    (gdb) p *((RelOptInfo *)root->join_rel_level[3]->head->next->next->data.ptr_value)->relids->words
    $57 = 70
    (gdb) p *((RelOptInfo *)root->join_rel_level[3]->head->next->next->next->data.ptr_value)->relids->words
    $58 = 134
    

[level=3]完成所有循环后的root->join_rel_level[3],构成连接的relids组合,一共20个(请参照数学组合的计算),包括1&2&4,1&2&5,1&2&6,1&2&7,1&4&5,1&4&6,1&4&7,...

    
    
    ...
    (gdb) p *root->join_rel_level[3]
    $68 = {type = T_List, length = 20, head = 0x2fd90d8, tail = 0x2f7f248}
    

[level=3]尝试bushy plans,达不到要求,退出循环

    
    
    142     for (k = 2;; k++)
    (gdb) 
    144         int         other_level = level - k;
    (gdb) 
    150         if (k > other_level)
    150         if (k > other_level)
    (gdb) n
    151             break;
    

[level=3]完成level=3的调用,开始level 4调用

    
    
    (gdb) 
    standard_join_search (root=0x3006e28, levels_needed=6, initial_rels=0x2fcaf18) at allpaths.c:2757
    2757            foreach(lc, root->join_rel_level[lev])
    (gdb) c
    Continuing.
    
    Breakpoint 1, join_search_one_level (root=0x3006e28, level=4) at joinrels.c:67
    67      List      **joinrels = root->join_rel_level;
    

[level=4]完成第一轮循环调用,查看root->join_rel_level[4],relids分别是54/86/150,即1&2&4&5,1&2&4&6,1&2&4&7

    
    
    ...
    89          {
    (gdb) 
    83      foreach(r, joinrels[level - 1])
    (gdb) p *root->join_rel_level[4]
    $69 = {type = T_List, length = 3, head = 0x2f838e0, tail = 0x30654d8}
    (gdb)  p *((RelOptInfo *)root->join_rel_level[4]->head->data.ptr_value)->relids->words
    $70 = 54
    (gdb)  p *((RelOptInfo *)root->join_rel_level[4]->head->next->data.ptr_value)->relids->words
    $71 = 86
    (gdb)  p *((RelOptInfo *)root->join_rel_level[4]->head->next->next->data.ptr_value)->relids->words
    $72 = 150
    

[level=4]所有循环后的root->join_rel_level[4],构成连接的relids组合,一共15个

    
    
    (gdb) b joinrels.c:142
    Breakpoint 2 at 0x75576a: file joinrels.c, line 142.
    (gdb) c
    Continuing.
    
    Breakpoint 2, join_search_one_level (root=0x3006e28, level=4) at joinrels.c:142
    142     for (k = 2;; k++)
    (gdb) p *root->join_rel_level[4]
    $73 = {type = T_List, length = 15, head = 0x2f838e0, tail = 0x307bd78}
    

[level=4]尝试bushy plans

    
    
    ...
    (gdb) p k
    $74 = 2
    (gdb) p other_level
    $75 = 2
    

[level=4]遍历k级关系,k=other_level,同一层次的rel,两两组合,即1&2,3&4等尝试两两配对连接

    
    
    (gdb) n
    153         foreach(r, joinrels[k])
    ...
    (gdb) 
    168             if (k == other_level)
    

[level=4]如relids=6和relids=48的两个关系

    
    
    177                 if (!bms_overlap(old_rel->relids, new_rel->relids))
    (gdb) 
    184                     if (have_relevant_joinclause(root, old_rel, new_rel) ||
    (gdb) p *old_rel->relids->words
    $78 = 6
    (gdb) p *new_rel->relids->words
    $79 = 48
    

[level=4]构造新的关系,但该关系无法通过合法连接形成或者已存在,因此没有对root->join_rel_level[4]有所影响(调用前后均为15个Node)

    
    
    (gdb) n
    187                         (void) make_join_rel(root, old_rel, new_rel);
    (gdb) 
    173             for_each_cell(r2, other_rels)
    (gdb) p *root->join_rel_level[4]
    $80 = {type = T_List, length = 15, head = 0x2f838e0, tail = 0x307bd78}
    

[level=4]完成bushy plans,root->join_rel_level[4]元素个数没有变化

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 3, join_search_one_level (root=0x3006e28, level=4) at joinrels.c:213
    213     if (joinrels[level] == NIL)
    (gdb) p *root->join_rel_level[4]
    $82 = {type = T_List, length = 15, head = 0x2f838e0, tail = 0x307bd78}
    

[level=5]进入level=5调用

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 1, join_search_one_level (root=0x3006e28, level=5) at joinrels.c:67
    67      List      **joinrels = root->join_rel_level;
    

[level=5]完成第一轮循环调用,查看root->join_rel_level[5],relids分别是118/182,即1&2&4&5&6,1&2&4&6&7

    
    
    (gdb) p *root->join_rel_level[5]
    $83 = {type = T_List, length = 2, head = 0x30931d0, tail = 0x3093dc8}
    (gdb) p *((RelOptInfo *)root->join_rel_level[5]->head->data.ptr_value)->relids->words
    $85 = 118
    (gdb) p *((RelOptInfo *)root->join_rel_level[5]->head->next->data.ptr_value)->relids->words
    $86 = 182
    

[level=5]所有循环后的root->join_rel_level[5],构成连接的relids组合,一共6个

    
    
    (gdb) p *root->join_rel_level[5]
    $87 = {type = T_List, length = 6, head = 0x30931d0, tail = 0x309d188}
    

[level=5]尝试bushy plans,即2个rels连接生成的关系 join 3个rels连接生成的关系  
完成调用

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 3, join_search_one_level (root=0x3006e28, level=5) at joinrels.c:213
    213     if (joinrels[level] == NIL)
    (gdb) p *root->join_rel_level[5]
    $91 = {type = T_List, length = 6, head = 0x30931d0, tail = 0x309d188}
    

[level=6]进入level=6调用

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 1, join_search_one_level (root=0x3006e28, level=6) at joinrels.c:67
    67      List      **joinrels = root->join_rel_level;
    

[level=6]与level=1的rels连接后,形成1个新的关系

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 2, join_search_one_level (root=0x3006e28, level=6) at joinrels.c:142
    142     for (k = 2;; k++)
    (gdb) p *root->join_rel_level[6]
    $92 = {type = T_List, length = 1, head = 0x3104cf8, tail = 0x3104cf8}
    

[level=6]尝试bushy plans,即2个rels连接生成的关系 join 4个rels连接生成的关系 & 3 join 3  
完成调用,生成level=6的结果链表

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 3, join_search_one_level (root=0x3006e28, level=6) at joinrels.c:213
    213     if (joinrels[level] == NIL)
    (gdb) p *root->join_rel_level[6]
    $93 = {type = T_List, length = 1, head = 0x3104cf8, tail = 0x3104cf8}
    (gdb) p *(RelOptInfo *)root->join_rel_level[6]->head->data.ptr_value
    $94 = {type = T_RelOptInfo, reloptkind = RELOPT_JOINREL, relids = 0x3099a80, rows = 2, consider_startup = false, 
      consider_param_startup = false, consider_parallel = true, reltarget = 0x3104a08, pathlist = 0x3104ec0, ppilist = 0x0, 
      partial_pathlist = 0x0, cheapest_startup_path = 0x0, cheapest_total_path = 0x0, cheapest_unique_path = 0x0, 
      cheapest_parameterized_paths = 0x0, direct_lateral_relids = 0x0, lateral_relids = 0x0, relid = 0, reltablespace = 0, 
      rtekind = RTE_JOIN, min_attr = 0, max_attr = 0, attr_needed = 0x0, attr_widths = 0x0, lateral_vars = 0x0, 
      lateral_referencers = 0x0, indexlist = 0x0, statlist = 0x0, pages = 0, tuples = 0, allvisfrac = 0, subroot = 0x0, 
      subplan_params = 0x0, rel_parallel_workers = -1, serverid = 0, userid = 0, useridiscurrent = false, fdwroutine = 0x0, 
      fdw_private = 0x0, unique_for_rels = 0x0, non_unique_for_rels = 0x0, baserestrictinfo = 0x0, baserestrictcost = {
        startup = 0, per_tuple = 0}, baserestrict_min_security = 4294967295, joininfo = 0x0, has_eclass_joins = false, 
      top_parent_relids = 0x0, part_scheme = 0x0, nparts = 0, boundinfo = 0x0, partit
    

[level=6]查看访问路径

    
    
    (gdb) set $roi=(RelOptInfo *)root->join_rel_level[6]->head->data.ptr_value
    (gdb) p *$roi->pathlist
    $97 = {type = T_List, length = 1, head = 0x3104ea0, tail = 0x3104ea0}
    (gdb) p *(Node *)$roi->pathlist->head->data.ptr_value
    $98 = {type = T_NestPath}
    (gdb) p *(NestPath *)$roi->pathlist->head->data.ptr_value
    $99 = {path = {type = T_NestPath, pathtype = T_NestLoop, parent = 0x31047f8, pathtarget = 0x3104a08, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 2, startup_cost = 101.1725, 
        total_cost = 2218.2350000000001, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
      outerjoinpath = 0x2fccd80, innerjoinpath = 0x3107820, joinrestrictinfo = 0x3107ae0}
    

该path的innerjoinpath(构造该连接inner关系的path)和outerjoinpath(构造该连接outer关系的path)

    
    
    (gdb) p *$np->innerjoinpath
    $109 = {type = T_MaterialPath, pathtype = T_Material, parent = 0x3077c70, pathtarget = 0x3077e80, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 5, startup_cost = 97.922499999999999, 
      total_cost = 2013.9974999999999, pathkeys = 0x0}
    (gdb) p *$np->outerjoinpath
    $110 = {type = T_HashPath, pathtype = T_HashJoin, parent = 0x2f54050, pathtarget = 0x2fcbf88, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100, startup_cost = 3.25, total_cost = 196.75, 
      pathkeys = 0x0}
    

DONE!

    
    
    (gdb) c
    Continuing.
    
    
    

### 三、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[cost.h](https://doxygen.postgresql.org/cost_8h_source.html)  
[costsize.c](https://doxygen.postgresql.org/costsize_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

