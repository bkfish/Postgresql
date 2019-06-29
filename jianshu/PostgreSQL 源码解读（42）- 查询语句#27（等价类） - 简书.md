上一小节介绍了函数query_planner中子函数子函数build_base_rel_tlists/find_placeholders_in_jointree/find_lateral_references的实现逻辑,本节介绍下一个子函数deconstruct_jointree函数中涉及的数学知识:等价类以及等价类在PG中的应用实例。

### 一、数学基础

**等价关系**  
等价关系（equivalence relation）即设 _R_ 是某个集合 _A_ 上的一个二元关系。若 _R_ 满足以下条件：  
1.自反性:∀x∈A,xRx  
2.对称性:∀x,y∈A,xRy ⇒ yRx  
3.传递性:∀x,y,z∈A,(xRy ∧ yRz) ⇒ xRz  
则称R是一个定义在A上的等价关系。习惯上会把等价关系的符号由R改写为∼  
详细的说明和例子详见[维基百科](https://zh.wikipedia.org/wiki/%E7%AD%89%E4%BB%B7%E5%85%B3%E7%B3%BB)

**等价类**  
假设在一个集合A上定义一个等价关系（用∼表示），则A中的某个元素x的等价类就是在A中等价于x的所有元素所形成的子集:  
[x] = {y∈A,y∼x}  
详细的说明和例子详见[维基百科](https://zh.wikipedia.org/wiki/%E7%AD%89%E4%BB%B7%E7%B1%BB)

### 二、应用

PG在函数deconstruct_jointree中对约束条件子句（qual
clauses）进行扫描,可能会在非外连接的连接条件中找到等式,如A=B,这时候会创建一个等价类（记作{A,B}）.如果在后面发现另一个等式,如B=C,则把C加到已存在的等价类{A,B}中,即{A,B,C}。通过这个步骤,就产生了一些等价类,这些等价类中任何一对成员的等值关系都可以作为约束或者连接的条件.  
这样做的好处,一是可以通过这样的简化,只需要验证部分等值条件，而不需要验证所有的等值条件,就可以去掉一些多余的等价类条件约束;二是从相反的方向上考量,优化器在准备一个约束或者连接条件列表的时候,检查每个等价类是否能够派生新的满足当前约束或者连接的隐式条件。例如在上面的例子中,可以依据等价类{A,B,C}产生一个A=C的条件,这样优化器就可以尝试探索新的优化连接路径。

例一:

    
    
    testdb=# explain verbose select t1.dwbh,t2.grbh
    testdb-# from t_dwxx t1,t_grxx t2
    testdb-# where t1.dwbh = t2.dwbh and t1.dwbh = '1001';
                                   QUERY PLAN                               
    ------------------------------------------------------------------------     Nested Loop  (cost=0.00..16.06 rows=2 width=76)
       Output: t1.dwbh, t2.grbh -- 连接,无需执行filter操作
       ->  Seq Scan on public.t_dwxx t1  (cost=0.00..1.04 rows=1 width=38)
             Output: t1.dwmc, t1.dwbh, t1.dwdz
             Filter: ((t1.dwbh)::text = '1001'::text) -- 连接前完成选择操作
       ->  Seq Scan on public.t_grxx t2  (cost=0.00..15.00 rows=2 width=76)
             Output: t2.dwbh, t2.grbh, t2.xm, t2.xb, t2.nl
             Filter: ((t2.dwbh)::text = '1001'::text) -- 连接前完成选择操作
    (8 rows)
    
    

约束条件:t1.dwbh = t2.dwbh and t1.dwbh =
'1001',其相应的等价类可以简略的理解为:{t1.dwbh,t2.dwbh,'1001'},根据传递性,那么可以推导出隐性约束条件:t2.dwbh='1001',在连接前把谓词t1.dwbh='1001'和t2.dwbh='1002'下推到基表扫描中,减少连接的元组数量,从而实现查询优化.从查询计划来看,PG实际也是这样处理的.  
下面的SQL语句则不具备以上条件,因此在连接过程中还需要执行join filter.

    
    
    testdb=# explain verbose select t1.dwbh,t2.grbh
    testdb-# from t_dwxx t1,t_grxx t2
    testdb-# where t1.dwbh = t2.dwbh and t1.dwdz like '广东省%';
                                    QUERY PLAN                                
    --------------------------------------------------------------------------     Nested Loop  (cost=0.00..20.04 rows=2 width=76)
       Output: t1.dwbh, t2.grbh
       Join Filter: ((t1.dwbh)::text = (t2.dwbh)::text)
       ->  Seq Scan on public.t_dwxx t1  (cost=0.00..1.04 rows=1 width=38)
             Output: t1.dwmc, t1.dwbh, t1.dwdz
             Filter: ((t1.dwdz)::text ~~ '广东省%'::text)
       ->  Seq Scan on public.t_grxx t2  (cost=0.00..14.00 rows=400 width=76)
             Output: t2.dwbh, t2.grbh, t2.xm, t2.xb, t2.nl
    (8 rows)
    
    

例二:  
测试脚本如下:

    
    
    testdb=# explain verbose select t1.dwbh,t2.grbh
    from t_dwxx t1,t_grxx t2
    where t1.dwbh = t2.dwbh 
    order by t2.dwbh;
                                        QUERY PLAN                                     
    -----------------------------------------------------------------------------------     Sort  (cost=20.18..20.19 rows=6 width=114)
       Output: t1.dwbh, t2.grbh, t2.dwbh
       Sort Key: t1.dwbh
       ->  Hash Join  (cost=1.07..20.10 rows=6 width=114)
             Output: t1.dwbh, t2.grbh, t2.dwbh
             Inner Unique: true
             Hash Cond: ((t2.dwbh)::text = (t1.dwbh)::text)
             ->  Seq Scan on public.t_grxx t2  (cost=0.00..14.00 rows=400 width=76)
                   Output: t2.dwbh, t2.grbh, t2.xm, t2.xb, t2.nl
             ->  Hash  (cost=1.03..1.03 rows=3 width=38)
                   Output: t1.dwbh
                   ->  Seq Scan on public.t_dwxx t1  (cost=0.00..1.03 rows=3 width=38)
                         Output: t1.dwbh
    (13 rows)
    

从执行计划来看,虽然明确指定按t2.dwbh进行排序,但实际上按t1.dwbh进行排序.原因是存在等价类{t1.dwbh,t2.dwbh},不管在t1.dwbh还是t2.dwbh上排序,结果都是一样的,但t1.dwbh上排序,成本更低,因此优先选择此Key.

值得注意的是:PG的等价类只有在有等式条件（不包括外连接）和排序的情况下才产生,作为优化的一个方向可以考虑存在不等式时,把谓词进行下推.

    
    
    testdb=# explain verbose select t1.dwbh,t2.grbh
    from t_dwxx t1,t_grxx t2
    where t1.dwbh = t2.dwbh and t1.dwbh > '1001';
                                    QUERY PLAN                                
    --------------------------------------------------------------------------     Nested Loop  (cost=0.00..20.04 rows=2 width=76)
       Output: t1.dwbh, t2.grbh
       Join Filter: ((t1.dwbh)::text = (t2.dwbh)::text) -- 这一步必不可少
       ->  Seq Scan on public.t_dwxx t1  (cost=0.00..1.04 rows=1 width=38)
             Output: t1.dwmc, t1.dwbh, t1.dwdz
             Filter: ((t1.dwbh)::text > '1001'::text) -- Filter过滤
       ->  Seq Scan on public.t_grxx t2  (cost=0.00..14.00 rows=400 width=76)
             Output: t2.dwbh, t2.grbh, t2.xm, t2.xb, t2.nl -- 全表扫描,可以考虑加入Filter
    (8 rows)
    

### 三、参考资料

[等价关系](https://zh.wikipedia.org/wiki/%E7%AD%89%E4%BB%B7%E5%85%B3%E7%B3%BB)  
[等价类](https://zh.wikipedia.org/wiki/%E7%AD%89%E4%BB%B7%E7%B1%BB)  
[关系代数](https://www.jianshu.com/p/096291013f89)  
[查询优化基础](https://www.jianshu.com/p/2d5269651f18)

附:query_planner中的代码片段

    
    
         //...
         /*
          * Examine the targetlist and join tree, adding entries to baserel
          * targetlists for all referenced Vars, and generating PlaceHolderInfo
          * entries for all referenced PlaceHolderVars.  Restrict and join clauses
          * are added to appropriate lists belonging to the mentioned relations. We
          * also build EquivalenceClasses for provably equivalent expressions. The
          * SpecialJoinInfo list is also built to hold information about join order
          * restrictions.  Finally, we form a target joinlist for make_one_rel() to
          * work from.
          */
         build_base_rel_tlists(root, tlist);//构建"base rels"的投影列
     
         find_placeholders_in_jointree(root);//处理jointree中的PHI
     
         find_lateral_references(root);//处理jointree中Lateral依赖
     
         joinlist = deconstruct_jointree(root);//分解jointree
    
         /*
          * Reconsider any postponed outer-join quals now that we have built up
          * equivalence classes.  (This could result in further additions or
          * mergings of classes.)
          */
         reconsider_outer_join_clauses(root);//已创建等价类,那么需要重新考虑被下推后处理的外连接表达式
     
         /*
          * If we formed any equivalence classes, generate additional restriction
          * clauses as appropriate.  (Implied join clauses are formed on-the-fly
          * later.)
          */
         generate_base_implied_equalities(root);//等价类构建后,生成因此外加的约束语句
     
         //...
    

