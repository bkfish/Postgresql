上一小节介绍了函数query_planner中子函数deconstruct_jointree的主要实现逻辑,调用该函数后,PlannerInfo中生成等价类的相关信息,本节结合实际的内存数据介绍相关数据结构。

### 一、数据结构

    
    
     /*
      * EquivalenceClasses
      *
      * Whenever we can determine that a mergejoinable equality clause A = B is
      * not delayed by any outer join, we create an EquivalenceClass containing
      * the expressions A and B to record this knowledge.  If we later find another
      * equivalence B = C, we add C to the existing EquivalenceClass; this may
      * require merging two existing EquivalenceClasses.  At the end of the qual
      * distribution process, we have sets of values that are known all transitively
      * equal to each other, where "equal" is according to the rules of the btree
      * operator family(s) shown in ec_opfamilies, as well as the collation shown
      * by ec_collation.  (We restrict an EC to contain only equalities whose
      * operators belong to the same set of opfamilies.  This could probably be
      * relaxed, but for now it's not worth the trouble, since nearly all equality
      * operators belong to only one btree opclass anyway.  Similarly, we suppose
      * that all or none of the input datatypes are collatable, so that a single
      * collation value is sufficient.)
      *
      * We also use EquivalenceClasses as the base structure for PathKeys, letting
      * us represent knowledge about different sort orderings being equivalent.
      * Since every PathKey must reference an EquivalenceClass, we will end up
      * with single-member EquivalenceClasses whenever a sort key expression has
      * not been equivalenced to anything else.  It is also possible that such an
      * EquivalenceClass will contain a volatile expression ("ORDER BY random()"),
      * which is a case that can't arise otherwise since clauses containing
      * volatile functions are never considered mergejoinable.  We mark such
      * EquivalenceClasses specially to prevent them from being merged with
      * ordinary EquivalenceClasses.  Also, for volatile expressions we have
      * to be careful to match the EquivalenceClass to the correct targetlist
      * entry: consider SELECT random() AS a, random() AS b ... ORDER BY b,a.
      * So we record the SortGroupRef of the originating sort clause.
      *
      * We allow equality clauses appearing below the nullable side of an outer join
      * to form EquivalenceClasses, but these have a slightly different meaning:
      * the included values might be all NULL rather than all the same non-null
      * values.  See src/backend/optimizer/README for more on that point.
      *
      * NB: if ec_merged isn't NULL, this class has been merged into another, and
      * should be ignored in favor of using the pointed-to class.
      */
     typedef struct EquivalenceClass
     {
         NodeTag     type;
     
         List       *ec_opfamilies;  /* btree操作符族(pg_opfamily)Oids,btree operator family OIDs */
         Oid         ec_collation;   /* 主要用于排序的规则,collation, if datatypes are collatable */
         List       *ec_members;     /* 等价类成员链表,list of EquivalenceMembers */
         List       *ec_sources;     /* 产生等价类的RestrictInfo链表,list of generating RestrictInfos */
         List       *ec_derives;     /* 衍生的RestrictInfo链表,list of derived RestrictInfos */
         Relids      ec_relids;      /* 出现在成员中的所有relids,all relids appearing in ec_members, except
                                      * for child members (see below) */
         bool        ec_has_const;   /* 成员中是否存在常量?any pseudoconstants in ec_members? */
         bool        ec_has_volatile;    /* 成员中是否存在易变表达式(如Random等),the (sole) member is a volatile expr */
         bool        ec_below_outer_join;    /* 等价类是否应用于外连接下层?equivalence applies below an OJ */
         bool        ec_broken;      /* 产生所需要的子句是否失败?failed to generate needed clauses? */
         Index       ec_sortref;     /* 源于排序子句的标志,originating sortclause label, or 0 */
         Index       ec_min_security;    /* 最小安全等级,minimum security_level in ec_sources */
         Index       ec_max_security;    /* 最大安全等级,maximum security_level in ec_sources */
         struct EquivalenceClass *ec_merged; /* 合并后的等价类,set if merged into another EC */
     } EquivalenceClass;
     
     /*
      * If an EC contains a const and isn't below-outer-join, any PathKey depending
      * on it must be redundant, since there's only one possible value of the key.
      */
     #define EC_MUST_BE_REDUNDANT(eclass)  \
         ((eclass)->ec_has_const && !(eclass)->ec_below_outer_join)
     
     /*
      * EquivalenceMember - one member expression of an EquivalenceClass
      *
      * em_is_child signifies that this element was built by transposing a member
      * for an appendrel parent relation to represent the corresponding expression
      * for an appendrel child.  These members are used for determining the
      * pathkeys of scans on the child relation and for explicitly sorting the
      * child when necessary to build a MergeAppend path for the whole appendrel
      * tree.  An em_is_child member has no impact on the properties of the EC as a
      * whole; in particular the EC's ec_relids field does NOT include the child
      * relation.  An em_is_child member should never be marked em_is_const nor
      * cause ec_has_const or ec_has_volatile to be set, either.  Thus, em_is_child
      * members are not really full-fledged members of the EC, but just reflections
      * or doppelgangers of real members.  Most operations on EquivalenceClasses
      * should ignore em_is_child members, and those that don't should test
      * em_relids to make sure they only consider relevant members.
      *
      * em_datatype is usually the same as exprType(em_expr), but can be
      * different when dealing with a binary-compatible opfamily; in particular
      * anyarray_ops would never work without this.  Use em_datatype when
      * looking up a specific btree operator to work with this expression.
      */
     typedef struct EquivalenceMember
     {
         NodeTag     type;
     
         Expr       *em_expr;        /* 该成员所代表的表达式,the expression represented */
         Relids      em_relids;      /* 出现在表达式中的relids,all relids appearing in em_expr */
         Relids      em_nullable_relids; /* 低层外连接nullable端的relids,nullable by lower outer joins */
         bool        em_is_const;    /* 常量?expression is pseudoconstant? */
         bool        em_is_child;    /* 子Relation的衍生版本?derived version for a child relation? */
         Oid         em_datatype;    /* 操作族使用到的数据类型,the "nominal type" used by the opfamily */
     } EquivalenceMember;
     
    

### 二、跟踪分析

启动gdb,跟踪:

    
    
    (gdb) b query_planner
    Breakpoint 3 at 0x7693b5: file planmain.c, line 57.
    

执行函数deconstruct_jointree,查看root结构

    
    
    156   joinlist = deconstruct_jointree(root);
    (gdb) 
    163   reconsider_outer_join_clauses(root);
    (gdb) p *root
    $4 = {type = T_PlannerInfo, parse = 0x2c53ad0, glob = 0x2c8bff8, query_level = 1, parent_root = 0x0, plan_params = 0x0, 
      outer_params = 0x0, simple_rel_array = 0x2c941f8, simple_rel_array_size = 6, simple_rte_array = 0x2c94248, 
      all_baserels = 0x0, nullable_baserels = 0x0, join_rel_list = 0x0, join_rel_hash = 0x0, join_rel_level = 0x0, 
      join_cur_level = 0, init_plans = 0x0, cte_plan_ids = 0x0, multiexpr_params = 0x0, eq_classes = 0x2c960b8, 
      canon_pathkeys = 0x0, left_join_clauses = 0x0, right_join_clauses = 0x0, full_join_clauses = 0x0, join_info_list = 0x0, 
      append_rel_list = 0x0, rowMarks = 0x0, placeholder_list = 0x0, fkey_list = 0x0, query_pathkeys = 0x0, 
      group_pathkeys = 0x0, window_pathkeys = 0x0, distinct_pathkeys = 0x0, sort_pathkeys = 0x0, part_schemes = 0x0, 
      initial_rels = 0x0, upper_rels = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}, upper_targets = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 
        0x0}, processed_tlist = 0x2c8e3d0, grouping_map = 0x0, minmax_aggs = 0x0, planner_cxt = 0x2b9fde0, 
      total_table_pages = 0, tuple_fraction = 0, limit_tuples = -1, qual_security_level = 0, inhTargetKind = INHKIND_NONE, 
      hasJoinRTEs = true, hasLateralRTEs = true, hasDeletedRTEs = false, hasHavingQual = false, hasPseudoConstantQuals = false, 
      hasRecursion = false, wt_param_id = -1, non_recursive_path = 0x0, curOuterRels = 0x0, curOuterParams = 0x0, 
      join_search_private = 0x0, partColsUpdated = false}
    

root->eq_classes是等价类链表,其中的元素是等价类

    
    
    (gdb) p *root->eq_classes
    $1 = {type = T_List, length = 2, head = 0x2c6daf8, tail = 0x2c6ddf8}
    (gdb) set $ec1=(EquivalenceClass *)root->eq_classes->head->data.ptr_value
    (gdb) set $ec2=(EquivalenceClass *)root->eq_classes->head->next->data.ptr_value
    (gdb) p *$ec1
    $4 = {type = T_EquivalenceClass, ec_opfamilies = 0x2c6d980, ec_collation = 100, ec_members = 0x2c6da58, 
      ec_sources = 0x2c6d9f0, ec_derives = 0x0, ec_relids = 0x2c6da20, ec_has_const = false, ec_has_volatile = false, 
      ec_below_outer_join = false, ec_broken = false, ec_sortref = 0, ec_min_security = 0, ec_max_security = 0, ec_merged = 0x0}
    (gdb) p *$ec2
    $5 = {type = T_EquivalenceClass, ec_opfamilies = 0x2c6dc30, ec_collation = 100, ec_members = 0x2c6dd58, 
      ec_sources = 0x2c6dca0, ec_derives = 0x0, ec_relids = 0x2c6dd20, ec_has_const = true, ec_has_volatile = false, 
      ec_below_outer_join = false, ec_broken = false, ec_sortref = 0, ec_min_security = 0, ec_max_security = 0, ec_merged = 0x0}
    (gdb) 
    

**第1个等价类信息**  
ec_opfamilies

    
    
    (gdb) p *$ec1->ec_opfamilies
    $6 = {type = T_OidList, length = 2, head = 0x2c6d960, tail = 0x2c6d9b0}
    (gdb) p $ec1->ec_opfamilies->head->data.oid_value
    $7 = 1994
    (gdb) p $ec1->ec_opfamilies->head->next->data.oid_value
    $8 = 2095
    (gdb) 
    

数据字典中相应的记录:

    
    
    testdb=# select * from pg_opfamily where oid=2095;
     opfmethod |     opfname      | opfnamespace | opfowner 
    -----------+------------------+--------------+----------           403 | text_pattern_ops |           11 |       10
    (1 row)
    
    testdb=# select * from pg_opfamily where oid=1994;
     opfmethod | opfname  | opfnamespace | opfowner 
    -----------+----------+--------------+----------           403 | text_ops |           11 |       10
    (1 row)
    

ec_members,共有2个元素  
第1个元素,是rtindex=3的RTE,属性编号为2的字段,即t_grxx.grbh

    
    
    (gdb) p *$ec1->ec_members
    $10 = {type = T_List, length = 2, head = 0x2c6da38, tail = 0x2c6dad8}
    (gdb) set $ec1_em1=(EquivalenceMember *)$ec1->ec_members->head->data.ptr_value
    (gdb) set $ec1_em2=(EquivalenceMember *)$ec1->ec_members->head->next->data.ptr_value
    (gdb) p *$ec1_em1
    $13 = {type = T_EquivalenceMember, em_expr = 0x2c69f88, em_relids = 0x2c6d770, em_nullable_relids = 0x0, 
      em_is_const = false, em_is_child = false, em_datatype = 25}
    (gdb) p *$ec1_em1
    $13 = {type = T_EquivalenceMember, em_expr = 0x2c69f88, em_relids = 0x2c6d770, em_nullable_relids = 0x0, 
      em_is_const = false, em_is_child = false, em_datatype = 25}
    (gdb) p *$ec1_em1->em_expr
    $14 = {type = T_RelabelType}
    (gdb) p *(RelabelType *)$ec1_em1->em_expr
    $15 = {xpr = {type = T_RelabelType}, arg = 0x2c69f38, resulttype = 25, resulttypmod = -1, resultcollid = 100, 
      relabelformat = COERCE_IMPLICIT_CAST, location = -1}
    (gdb) p *((RelabelType *)$ec1_em1->em_expr)->arg
    $16 = {type = T_Var}
    (gdb) p *(Var *)((RelabelType *)$ec1_em1->em_expr)->arg
    $17 = {xpr = {type = T_Var}, varno = 3, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 3, varoattno = 2, location = 136} 
    

第2个元素,是rtindex=4的RTE,属性编号为1的字段,即t_jfxx.grbh

    
    
    (gdb) p *$ec1_em2->em_expr
    $28 = {type = T_RelabelType}
    (gdb) p *(RelabelType *)$ec1_em2->em_expr
    $29 = {xpr = {type = T_RelabelType}, arg = 0x2c69fd8, resulttype = 25, resulttypmod = -1, resultcollid = 100, 
      relabelformat = COERCE_IMPLICIT_CAST, location = -1}
    (gdb) p *((RelabelType *)$ec1_em2->em_expr)->arg
    $30 = {type = T_Var}
    (gdb) p *(Var *)((RelabelType *)$ec1_em2->em_expr)->arg
    $31 = {xpr = {type = T_Var}, varno = 4, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 4, varoattno = 1, location = 146}
    

其他信息

    
    
    (gdb) p *$ec1->ec_sources
    $34 = {type = T_List, length = 1, head = 0x2c6d9d0, tail = 0x2c6d9d0}
    (gdb) p *(Node *)$ec1->ec_sources->head->data.ptr_value
    $35 = {type = T_RestrictInfo}
    (gdb) p *(RestrictInfo *)$ec1->ec_sources->head->data.ptr_value
    $36 = {type = T_RestrictInfo, clause = 0x2c6a098, is_pushed_down = true, outerjoin_delayed = false, can_join = true, 
      pseudoconstant = false, leakproof = false, security_level = 0, clause_relids = 0x2c6d7a0, required_relids = 0x2c6d758, 
      outer_relids = 0x0, nullable_relids = 0x0, left_relids = 0x2c6d770, right_relids = 0x2c6d788, orclause = 0x0, 
      parent_ec = 0x0, eval_cost = {startup = -1, per_tuple = 0}, norm_selec = -1, outer_selec = -1, 
      mergeopfamilies = 0x2c6d980, left_ec = 0x2c6ce68, right_ec = 0x2c6ce68, left_em = 0x2c6d890, right_em = 0x2c6da88, 
      scansel_cache = 0x0, outer_is_left = false, hashjoinoperator = 0, left_bucketsize = -1, right_bucketsize = -1, 
      left_mcvfreq = -1, right_mcvfreq = -1}
    (gdb) p *$ec1->ec_relids
    $38 = {nwords = 1, words = 0x2c6da24}
    #即3号和4号RTE
    (gdb) p $ec1->ec_relids->words[0]
    $39 = 24  
    

**第2个等价类信息**

    
    
    (gdb) p *$ec2
    $41 = {type = T_EquivalenceClass, ec_opfamilies = 0x2c6dc30, ec_collation = 100, ec_members = 0x2c6dd58, 
      ec_sources = 0x2c6dca0, ec_derives = 0x0, ec_relids = 0x2c6dd20, ec_has_const = true, ec_has_volatile = false, 
      ec_below_outer_join = false, ec_broken = false, ec_sortref = 0, ec_min_security = 0, ec_max_security = 0, ec_merged = 0x0}
    

ec_opfamilies,与第1个等价类的信息一致

    
    
    (gdb) p *$ec2->ec_opfamilies
    $42 = {type = T_OidList, length = 2, head = 0x2c6dc60, tail = 0x2c6dc10}
    (gdb) p $ec2->ec_opfamilies->head->data.oid_value
    $43 = 1994
    (gdb) p $ec2->ec_opfamilies->head->next->data.oid_value
    $44 = 2095
    

ec_members,有3个元素

    
    
    (gdb) p *$ec2->ec_members
    $46 = {type = T_List, length = 3, head = 0x2c6dd38, tail = 0x2c6df20}
    (gdb) set $ec2_em1=(EquivalenceMember *)$ec2->ec_members->head->data.ptr_value
    (gdb) set $ec2_em2=(EquivalenceMember *)$ec2->ec_members->head->next->data.ptr_value
    (gdb) set $ec2_em3=(EquivalenceMember *)$ec2->ec_members->head->next->next->data.ptr_value
    
    

第1个元素,3号RTE,属性编号为1的字段,即t_grxx.dwbh

    
    
    (gdb) p *$ec2_em1
    $47 = {type = T_EquivalenceMember, em_expr = 0x2c69d58, em_relids = 0x2c6dbc8, em_nullable_relids = 0x0, 
      em_is_const = false, em_is_child = false, em_datatype = 25}
    (gdb) p *$ec2_em1->em_expr
    $48 = {type = T_RelabelType}
    (gdb) p *(RelabelType *)$ec2_em1->em_expr
    $49 = {xpr = {type = T_RelabelType}, arg = 0x2c69d08, resulttype = 25, resulttypmod = -1, resultcollid = 100, 
      relabelformat = COERCE_IMPLICIT_CAST, location = -1}
    (gdb) p *(Var *)((RelabelType *)$ec2_em1->em_expr)->arg
    $50 = {xpr = {type = T_Var}, varno = 3, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 3, varoattno = 1, location = 115}
    

第2个元素

    
    
    (gdb) p *$ec2_em2,1号RTE,属性编号为2的字段,即t_dwxx.dwbh
    $52 = {type = T_EquivalenceMember, em_expr = 0x2c69e28, em_relids = 0x2c6dbe0, em_nullable_relids = 0x0, 
      em_is_const = false, em_is_child = false, em_datatype = 25}
    (gdb) p *$ec2_em2->em_expr
    $53 = {type = T_RelabelType}
    (gdb) p *(Var *)((RelabelType *)$ec2_em2->em_expr)->arg
    $54 = {xpr = {type = T_Var}, varno = 1, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, varlevelsup = 0, 
      varnoold = 1, varoattno = 2, location = 125}
    

第3个元素,是一个常量,即'1001'

    
    
    (gdb) p *$ec2_em3
    $55 = {type = T_EquivalenceMember, em_expr = 0x2c6a498, em_relids = 0x0, em_nullable_relids = 0x0, em_is_const = true, 
      em_is_child = false, em_datatype = 25}
    (gdb) p *$ec2_em3->em_expr
    $56 = {type = T_Const}
    (gdb) p *((Const *)$ec2_em2->em_expr)->arg
    (gdb) p *(Const *)$ec2_em3->em_expr
    $58 = {xpr = {type = T_Const}, consttype = 25, consttypmod = -1, constcollid = 100, constlen = -1, constvalue = 46517720, 
      constisnull = false, constbyval = false, location = 172}
    

### 三、参考资料

[relation.h](https://doxygen.postgresql.org/relation_8h_source.html)

