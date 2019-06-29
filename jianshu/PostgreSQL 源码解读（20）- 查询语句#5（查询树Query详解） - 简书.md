本文简单介绍了PG查询优化重写后生成的查询树Query的详细结构，查询重写优化的输入是上一节介绍的解析树Parsetree。

### 一、查询树结构

查询语句：

    
    
    testdb=# select * from (
    testdb(# select t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je
    testdb(# from t_dwxx inner join t_grxx on t_dwxx.dwbh = t_grxx.dwbh
    testdb(# inner join t_jfxx on t_grxx.grbh = t_jfxx.grbh
    testdb(# where t_dwxx.dwbh IN ('1001')
    testdb(# union all
    testdb(# select t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je
    testdb(# from t_dwxx inner join t_grxx on t_dwxx.dwbh = t_grxx.dwbh
    testdb(# inner join t_jfxx on t_grxx.grbh = t_jfxx.grbh
    testdb(# where t_dwxx.dwbh IN ('1002') 
    testdb(# ) as ret
    testdb-# order by ret.grbh
    testdb-# limit 4;
    

跟踪分析：

    
    
    (gdb) b exec_simple_query
    Breakpoint 1 at 0x84cad8: file postgres.c, line 893.
    (gdb) c
    Continuing.
    1047            querytree_list = pg_analyze_and_rewrite(parsetree, query_string,
    (gdb) 
    1050            plantree_list = pg_plan_queries(querytree_list,
    (gdb) p *querytree_list
    $1 = {type = T_List, length = 1, head = 0x170c858, tail = 0x170c858}
    (gdb)  p *(Node *)(querytree_list->head->data.ptr_value)
    $2 = {type = T_Query}
    (gdb) p *(Query *)(querytree_list->head->data.ptr_value)
    $3 = {type = T_Query, commandType = CMD_SELECT, querySource = QSRC_ORIGINAL, queryId = 0, canSetTag = true, 
      utilityStmt = 0x0, resultRelation = 0, hasAggs = false, hasWindowFuncs = false, hasTargetSRFs = false, 
      hasSubLinks = false, hasDistinctOn = false, hasRecursive = false, hasModifyingCTE = false, 
      hasForUpdate = false, hasRowSecurity = false, cteList = 0x0, rtable = 0x170be68, jointree = 0x170c7d8, 
      targetList = 0x170c428, override = OVERRIDING_NOT_SET, onConflict = 0x0, returningList = 0x0, 
      groupClause = 0x0, groupingSets = 0x0, havingQual = 0x0, windowClause = 0x0, distinctClause = 0x0, 
      sortClause = 0x170c6b8, limitOffset = 0x0, limitCount = 0x170c788, rowMarks = 0x0, setOperations = 0x0, 
      constraintDeps = 0x0, withCheckOptions = 0x0, stmt_location = 0, stmt_len = 455}
    (gdb) set $query=(Query *)(querytree_list->head->data.ptr_value)
    

**Query- >rtable**

    
    
    #---------------------->rtable
    (gdb) set $rtable=$query->rtable
    (gdb) p *$rtable
    $8 = {type = T_List, length = 3, head = 0x170be48, tail = 0x170f6b0}
    (gdb) p *(Node *)($rtable->head->data.ptr_value)
    $9 = {type = T_RangeTblEntry}
    (gdb) p *(RangeTblEntry *)($rtable->head->data.ptr_value)
    $10 = {type = T_RangeTblEntry, rtekind = RTE_SUBQUERY, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x1667500, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, 
      functions = 0x0, funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, 
      ctelevelsup = 0, self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, 
      enrname = 0x0, enrtuples = 0, alias = 0x1666d40, eref = 0x170bc18, lateral = false, inh = true, 
      inFromCl = true, requiredPerms = 2, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, 
      updatedCols = 0x0, securityQuals = 0x0}
    (gdb) set $rte=(RangeTblEntry *)($rtable->head->data.ptr_value)
    (gdb) p *$rte->subquery
    $12 = {type = T_Query, commandType = CMD_SELECT, querySource = QSRC_ORIGINAL, queryId = 0, canSetTag = true, 
      utilityStmt = 0x0, resultRelation = 0, hasAggs = false, hasWindowFuncs = false, hasTargetSRFs = false, 
      hasSubLinks = false, hasDistinctOn = false, hasRecursive = false, hasModifyingCTE = false, 
      hasForUpdate = false, hasRowSecurity = false, cteList = 0x0, rtable = 0x16fe4e8, jointree = 0x170bbe8, 
      targetList = 0x170b358, override = OVERRIDING_NOT_SET, onConflict = 0x0, returningList = 0x0, 
      groupClause = 0x0, groupingSets = 0x0, havingQual = 0x0, windowClause = 0x0, distinctClause = 0x0, 
      sortClause = 0x0, limitOffset = 0x0, limitCount = 0x0, rowMarks = 0x0, setOperations = 0x1667610, 
      constraintDeps = 0x0, withCheckOptions = 0x0, stmt_location = 0, stmt_len = 0}
    (gdb) p *$rte->alias
    $13 = {type = T_Alias, aliasname = 0x1666d28 "ret", colnames = 0x0}
    (gdb) p *$rte->eref
    $14 = {type = T_Alias, aliasname = 0x170bc48 "ret", colnames = 0x170bcb8}
    (gdb) p *$rte->eref->colnames
    $15 = {type = T_List, length = 5, head = 0x170bc98, tail = 0x170be28}
    (gdb) p *(Node *)$rte->eref->colnames->head->data.ptr_value
    $16 = {type = T_String}
    (gdb) p *(Value *)$rte->eref->colnames->head->data.ptr_value
    $17 = {type = T_String, val = {ival = 24165472, str = 0x170bc60 "dwmc"}}
    ---->subquery
    (gdb) p *(RangeTblEntry *)$rte->subquery->rtable->head->data.ptr_value
    $26 = {type = T_RangeTblEntry, rtekind = RTE_SUBQUERY, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x16faf98, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, 
      functions = 0x0, funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, 
      ctelevelsup = 0, self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, 
      enrname = 0x0, enrtuples = 0, alias = 0x16fe240, eref = 0x16fe290, lateral = false, inh = false, 
      inFromCl = false, requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, 
      updatedCols = 0x0, securityQuals = 0x0}
    (gdb) set $rte_sq_rte=((RangeTblEntry *)$rte->subquery->rtable->head->data.ptr_value)
    (gdb) p *(RangeTblEntry *)$rte_sq_rte->subquery->rtable->head->data.ptr_value
    $30 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26754, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x16677c0, lateral = false, inh = true, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x16fbda8, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) set $rte_sq_rte_sq_rte=(RangeTblEntry *)$rte_sq_rte->subquery->rtable->head->data.ptr_value
    (gdb) p *$rte_sq_rte_sq_rte
    $42 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26754, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x16677c0, lateral = false, inh = true, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x16fbda8, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) set $rte_sq_rte_sq_rtable=$rte_sq_rte->subquery->rtable
    (gdb) p *(RangeTblEntry *)($rte_sq_rte_sq_rtable->head->data.ptr_value)
    $60 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26754, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x16677c0, lateral = false, inh = true, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x16fbda8, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *(RangeTblEntry *)($rte_sq_rte_sq_rtable->head->next->data.ptr_value)
    $61 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26757, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x16fb4f0, lateral = false, inh = true, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x16fbe10, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *(RangeTblEntry *)($rte_sq_rte_sq_rtable->head->next->next->data.ptr_value)
    $62 = {type = T_RangeTblEntry, rtekind = RTE_JOIN, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x16fbff8, 
      functions = 0x0, funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, 
      ctelevelsup = 0, self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, 
      enrname = 0x0, enrtuples = 0, alias = 0x0, eref = 0x16fc318, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *(RangeTblEntry *)($rte_sq_rte_sq_rtable->head->next->next->next->data.ptr_value)
    $63 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26760, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x16fc678, lateral = false, inh = true, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x16fd1d0, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *(RangeTblEntry *)($rte_sq_rte_sq_rtable->tail->data.ptr_value)
    $64 = {type = T_RangeTblEntry, rtekind = RTE_JOIN, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x16fd3b8, 
      functions = 0x0, funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, 
      ctelevelsup = 0, self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, 
      enrname = 0x0, enrtuples = 0, alias = 0x0, eref = 0x16fd798, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    
    
    (gdb) p *(FromExpr *)($rte_sq_rte->subquery->jointree)
    $44 = {type = T_FromExpr, fromlist = 0x16fda18, quals = 0x16fe0f0}
    (gdb) set $rtesq2_jointree=(FromExpr *)($rte_sq_rte->subquery->jointree)
    (gdb) p *$rtesq2_jointree->fromlist
    $48 = {type = T_List, length = 1, head = 0x16fd9f8, tail = 0x16fd9f8}
    (gdb) p *(Node *)$rtesq2_jointree->fromlist->head->data.ptr_value
    $49 = {type = T_JoinExpr}
    (gdb) set $tmpvar=(JoinExpr *)$rtesq2_jointree->fromlist->head->data.ptr_value
    (gdb) p *$tmpvar
    $3 = {type = T_JoinExpr, jointype = JOIN_INNER, isNatural = false, larg = 0x2b68730, rarg = 0x2c215e8, usingClause = 0x0, quals = 0x2c28130, alias = 0x0, rtindex = 5}
    (gdb) p *$tmpvar->larg
    $4 = {type = T_JoinExpr}
    (gdb) p *(JoinExpr *)$tmpvar->larg
    $5 = {type = T_JoinExpr, jointype = JOIN_INNER, isNatural = false, larg = 0x2c1e848, rarg = 0x2c1ebd8, 
      usingClause = 0x0, quals = 0x2c20c48, alias = 0x0, rtindex = 3}
    (gdb) p *(JoinExpr *)$tmpvar->rarg
    $6 = {type = T_RangeTblRef, jointype = JOIN_SEMI, isNatural = 8, larg = 0x2b66de0, rarg = 0x636d7764, 
      usingClause = 0x10, quals = 0x2b66de0, alias = 0xda, rtindex = 46274048}
    (gdb) p *(JoinExpr *)$tmpvar->quals
    $7 = {type = T_OpExpr, jointype = 98, isNatural = 67, larg = 0x0, rarg = 0x64, usingClause = 0x2c27fb8, 
      quals = 0xa9, alias = 0x0, rtindex = 0}
    (gdb) p *((JoinExpr *)$tmpvar->larg)->larg
    $8 = {type = T_RangeTblRef}
    (gdb) p *(RangeTblRef *)((JoinExpr *)$tmpvar->larg)->larg
    $9 = {type = T_RangeTblRef, rtindex = 1}
    (gdb) p *(RangeTblRef *)((JoinExpr *)$tmpvar->larg)->rarg
    $10 = {type = T_RangeTblRef, rtindex = 2}
    (gdb) p *(RangeTblRef *)((JoinExpr *)$tmpvar->larg)->quals
    $11 = {type = T_OpExpr, rtindex = 98}
    (gdb) p *(Node *)((JoinExpr *)$tmpvar->larg)->quals
    $12 = {type = T_OpExpr}
    (gdb) p *(OpExpr *)((JoinExpr *)$tmpvar->larg)->quals
    $13 = {xpr = {type = T_OpExpr}, opno = 98, opfuncid = 67, opresulttype = 16, opretset = false, opcollid = 0, 
      inputcollid = 100, args = 0x2c20b88, location = 122}
      (gdb) p *((OpExpr *)((JoinExpr *)$tmpvar->larg)->quals)->args
    $72 = {type = T_List, length = 2, head = 0x2c20bd8, tail = 0x2c20bb8}
    (gdb) p *(Node *)((OpExpr *)((JoinExpr *)$tmpvar->larg)->quals)->args->head->data.ptr_value
    $73 = {type = T_RelabelType}
    (gdb) p *(RelabelType *)((OpExpr *)((JoinExpr *)$tmpvar->larg)->quals)->args->head->data.ptr_value
    $74 = {xpr = {type = T_RelabelType}, arg = 0x2c1f1d8, resulttype = 25, resulttypmod = -1, resultcollid = 100, 
      relabelformat = COERCE_IMPLICIT_CAST, location = -1}
    (gdb) p *((RelabelType *)((OpExpr *)((JoinExpr *)$tmpvar->larg)->quals)->args->head->data.ptr_value))->arg
    Junk after end of expression.
    (gdb) p *((RelabelType *)((OpExpr *)((JoinExpr *)$tmpvar->larg)->quals)->args->head->data.ptr_value)->arg
    $75 = {type = T_Var}
    (gdb) p *(Var *)((RelabelType *)((OpExpr *)((JoinExpr *)$tmpvar->larg)->quals)->args->head->data.ptr_value)->arg
    $76 = {xpr = {type = T_Var}, varno = 1, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, 
      varlevelsup = 0, varnoold = 1, varoattno = 2, location = 110}
    (gdb) p *(Var *)((RelabelType *)((OpExpr *)((JoinExpr *)$tmpvar->larg)->quals)->args->tail->data.ptr_value)->arg
    $77 = {xpr = {type = T_Var}, varno = 2, varattno = 1, vartype = 1043, vartypmod = 14, varcollid = 100, 
      varlevelsup = 0, varnoold = 2, varoattno = 1, location = 124}
    (gdb) p *(RangeTblRef *)((JoinExpr *)$tmpvar)->rarg
    $24 = {type = T_RangeTblRef, rtindex = 4}
    (gdb) set $rtesq2_targetList=($rte_sq_rte->subquery->targetList)
    (gdb) p *$rtesq2_targetList
    $30 = {type = T_List, length = 5, head = 0x2c28840, tail = 0x2c28b70}
    (gdb) p *(Node *)$rtesq2_targetList->head->data.ptr_value
    $32 = {type = T_TargetEntry}
    (gdb) p *(TargetEntry *)$rtesq2_targetList->head->data.ptr_value
    $33 = {xpr = {type = T_TargetEntry}, expr = 0x2c28770, resno = 1, resname = 0x2b67be8 "dwmc", 
      ressortgroupref = 0, resorigtbl = 26754, resorigcol = 1, resjunk = false}
    (gdb) p *$rte->subquery->rtable
    $34 = {type = T_List, length = 2, head = 0x2c2a200, tail = 0x2c2d5f8}
    (gdb) set $rte_sq_rte2=((RangeTblEntry *)$rte->subquery->rtable->head->next->data.ptr_value)
    #分析过程类似于rte1
    (gdb) p *$rte_sq_rte2
    $35 = {type = T_RangeTblEntry, rtekind = RTE_SUBQUERY, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x2c211b8, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, 
      functions = 0x0, funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, 
      ctelevelsup = 0, self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, 
      enrname = 0x0, enrtuples = 0, alias = 0x2c2d370, eref = 0x2c2d3c0, lateral = false, inh = false, 
      inFromCl = false, requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, 
      updatedCols = 0x0, securityQuals = 0x0}
    (gdb) 
    (gdb) p *(FromExpr *)$rte->subquery->jointree
    $41 = {type = T_FromExpr, fromlist = 0x0, quals = 0x0}
    (gdb) p *$rte->subquery->targetList
    $45 = {type = T_List, length = 5, head = 0x2c2ddb8, tail = 0x2c2e328}
    (gdb) p *(Node *)$rte->subquery->targetList->head->data.ptr_value
    $46 = {type = T_TargetEntry}
    (gdb) p *(TargetEntry *)$rte->subquery->targetList->head->data.ptr_value
    $47 = {xpr = {type = T_TargetEntry}, expr = 0x2c2dd18, resno = 1, resname = 0x2c2dd00 "dwmc", 
      ressortgroupref = 0, resorigtbl = 0, resorigcol = 0, resjunk = false}
    
    

**Query- >jointree**

    
    
    #---------------------->jointree
    (gdb) p *$query->jointree
    $50 = {type = T_FromExpr, fromlist = 0x2c2e9c0, quals = 0x0}
    (gdb) p *$query->jointree->fromlist
    $51 = {type = T_List, length = 1, head = 0x2c2e9a0, tail = 0x2c2e9a0}
    (gdb) p *(Node *)$query->jointree->fromlist->head->data.ptr_value
    $52 = {type = T_RangeTblRef}
    (gdb) 
    
    

**Query- >targetList**

    
    
    #---------------------->targetList
    (gdb) p *$query->targetList
    $54 = {type = T_List, length = 5, head = 0x2c2ee88, tail = 0x2c2f078}
    (gdb) p *(TargetEntry *)$query->targetList->head->data.ptr_value
    $55 = {xpr = {type = T_TargetEntry}, expr = 0x2c2ea78, resno = 1, resname = 0x2c2e9f0 "dwmc", 
      ressortgroupref = 0, resorigtbl = 0, resorigcol = 0, resjunk = false}
    (gdb) 
    (gdb)
    

**Query- >sortClause**

    
    
    #---------------------->sortClause
    (gdb) p *(Node *)$query->sortClause->head->data.ptr_value
    $57 = {type = T_SortGroupClause}
    (gdb) p *(SortGroupClause *)$query->sortClause->head->data.ptr_value
    $58 = {type = T_SortGroupClause, tleSortGroupRef = 1, eqop = 98, sortop = 664, nulls_first = false, 
      hashable = true}
    

**Query- >limitCount**

    
    
    #---------------------->limitCount
    (gdb) p *$query->limitCount
    $59 = {type = T_FuncExpr}
    (gdb) p *(FuncExpr *)$query->limitCount
    $60 = {xpr = {type = T_FuncExpr}, funcid = 481, funcresulttype = 20, funcretset = false, 
      funcvariadic = false, funcformat = COERCE_IMPLICIT_CAST, funccollid = 0, inputcollid = 0, args = 0x2c2fa68, 
      location = -1}
    (gdb) 
    
    

SQL语句最终的查询树结构如下:

  

![](https://upload-images.jianshu.io/upload_images/8194836-7f210882c7e217c5.png)

查询树

注意：RangeTblRef中的rtindex指向的是rtable链表RangeTblEntry的Index。

### 二、数据结构

**RangeTblEntry**

    
    
    /*--------------------      * RangeTblEntry -      *    A range table is a List of RangeTblEntry nodes.
      *
      *    A range table entry may represent a plain relation, a sub-select in
      *    FROM, or the result of a JOIN clause.  (Only explicit JOIN syntax
      *    produces an RTE, not the implicit join resulting from multiple FROM
      *    items.  This is because we only need the RTE to deal with SQL features
      *    like outer joins and join-output-column aliasing.)  Other special
      *    RTE types also exist, as indicated by RTEKind.
      *
      *    Note that we consider RTE_RELATION to cover anything that has a pg_class
      *    entry.  relkind distinguishes the sub-cases.
      *
      *    alias is an Alias node representing the AS alias-clause attached to the
      *    FROM expression, or NULL if no clause.
      *
      *    eref is the table reference name and column reference names (either
      *    real or aliases).  Note that system columns (OID etc) are not included
      *    in the column list.
      *    eref->aliasname is required to be present, and should generally be used
      *    to identify the RTE for error messages etc.
      *
      *    In RELATION RTEs, the colnames in both alias and eref are indexed by
      *    physical attribute number; this means there must be colname entries for
      *    dropped columns.  When building an RTE we insert empty strings ("") for
      *    dropped columns.  Note however that a stored rule may have nonempty
      *    colnames for columns dropped since the rule was created (and for that
      *    matter the colnames might be out of date due to column renamings).
      *    The same comments apply to FUNCTION RTEs when a function's return type
      *    is a named composite type.
      *
      *    In JOIN RTEs, the colnames in both alias and eref are one-to-one with
      *    joinaliasvars entries.  A JOIN RTE will omit columns of its inputs when
      *    those columns are known to be dropped at parse time.  Again, however,
      *    a stored rule might contain entries for columns dropped since the rule
      *    was created.  (This is only possible for columns not actually referenced
      *    in the rule.)  When loading a stored rule, we replace the joinaliasvars
      *    items for any such columns with null pointers.  (We can't simply delete
      *    them from the joinaliasvars list, because that would affect the attnums
      *    of Vars referencing the rest of the list.)
      *
      *    inh is true for relation references that should be expanded to include
      *    inheritance children, if the rel has any.  This *must* be false for
      *    RTEs other than RTE_RELATION entries.
      *
      *    inFromCl marks those range variables that are listed in the FROM clause.
      *    It's false for RTEs that are added to a query behind the scenes, such
      *    as the NEW and OLD variables for a rule, or the subqueries of a UNION.
      *    This flag is not used anymore during parsing, since the parser now uses
      *    a separate "namespace" data structure to control visibility, but it is
      *    needed by ruleutils.c to determine whether RTEs should be shown in
      *    decompiled queries.
      *
      *    requiredPerms and checkAsUser specify run-time access permissions
      *    checks to be performed at query startup.  The user must have *all*
      *    of the permissions that are OR'd together in requiredPerms (zero
      *    indicates no permissions checking).  If checkAsUser is not zero,
      *    then do the permissions checks using the access rights of that user,
      *    not the current effective user ID.  (This allows rules to act as
      *    setuid gateways.)  Permissions checks only apply to RELATION RTEs.
      *
      *    For SELECT/INSERT/UPDATE permissions, if the user doesn't have
      *    table-wide permissions then it is sufficient to have the permissions
      *    on all columns identified in selectedCols (for SELECT) and/or
      *    insertedCols and/or updatedCols (INSERT with ON CONFLICT DO UPDATE may
      *    have all 3).  selectedCols, insertedCols and updatedCols are bitmapsets,
      *    which cannot have negative integer members, so we subtract
      *    FirstLowInvalidHeapAttributeNumber from column numbers before storing
      *    them in these fields.  A whole-row Var reference is represented by
      *    setting the bit for InvalidAttrNumber.
      *
      *    securityQuals is a list of security barrier quals (boolean expressions),
      *    to be tested in the listed order before returning a row from the
      *    relation.  It is always NIL in parser output.  Entries are added by the
      *    rewriter to implement security-barrier views and/or row-level security.
      *    Note that the planner turns each boolean expression into an implicitly
      *    AND'ed sublist, as is its usual habit with qualification expressions.
      *--------------------      */
     typedef enum RTEKind
     {
         RTE_RELATION,               /* ordinary relation reference */
         RTE_SUBQUERY,               /* subquery in FROM */
         RTE_JOIN,                   /* join */
         RTE_FUNCTION,               /* function in FROM */
         RTE_TABLEFUNC,              /* TableFunc(.., column list) */
         RTE_VALUES,                 /* VALUES (<exprlist>), (<exprlist>), ... */
         RTE_CTE,                    /* common table expr (WITH list element) */
         RTE_NAMEDTUPLESTORE         /* tuplestore, e.g. for AFTER triggers */
     } RTEKind;
     
     typedef struct RangeTblEntry
     {
         NodeTag     type;
     
         RTEKind     rtekind;        /* see above */
     
         /*
          * XXX the fields applicable to only some rte kinds should be merged into
          * a union.  I didn't do this yet because the diffs would impact a lot of
          * code that is being actively worked on.  FIXME someday.
          */
     
         /*
          * Fields valid for a plain relation RTE (else zero):
          *
          * As a special case, RTE_NAMEDTUPLESTORE can also set relid to indicate
          * that the tuple format of the tuplestore is the same as the referenced
          * relation.  This allows plans referencing AFTER trigger transition
          * tables to be invalidated if the underlying table is altered.
          */
         Oid         relid;          /* OID of the relation */
         char        relkind;        /* relation kind (see pg_class.relkind) */
         struct TableSampleClause *tablesample;  /* sampling info, or NULL */
     
         /*
          * Fields valid for a subquery RTE (else NULL):
          */
         Query      *subquery;       /* the sub-query */
         bool        security_barrier;   /* is from security_barrier view? */
     
         /*
          * Fields valid for a join RTE (else NULL/zero):
          *
          * joinaliasvars is a list of (usually) Vars corresponding to the columns
          * of the join result.  An alias Var referencing column K of the join
          * result can be replaced by the K'th element of joinaliasvars --- but to
          * simplify the task of reverse-listing aliases correctly, we do not do
          * that until planning time.  In detail: an element of joinaliasvars can
          * be a Var of one of the join's input relations, or such a Var with an
          * implicit coercion to the join's output column type, or a COALESCE
          * expression containing the two input column Vars (possibly coerced).
          * Within a Query loaded from a stored rule, it is also possible for
          * joinaliasvars items to be null pointers, which are placeholders for
          * (necessarily unreferenced) columns dropped since the rule was made.
          * Also, once planning begins, joinaliasvars items can be almost anything,
          * as a result of subquery-flattening substitutions.
          */
         JoinType    jointype;       /* type of join */
         List       *joinaliasvars;  /* list of alias-var expansions */
     
         /*
          * Fields valid for a function RTE (else NIL/zero):
          *
          * When funcordinality is true, the eref->colnames list includes an alias
          * for the ordinality column.  The ordinality column is otherwise
          * implicit, and must be accounted for "by hand" in places such as
          * expandRTE().
          */
         List       *functions;      /* list of RangeTblFunction nodes */
         bool        funcordinality; /* is this called WITH ORDINALITY? */
     
         /*
          * Fields valid for a TableFunc RTE (else NULL):
          */
         TableFunc  *tablefunc;
     
         /*
          * Fields valid for a values RTE (else NIL):
          */
         List       *values_lists;   /* list of expression lists */
     
         /*
          * Fields valid for a CTE RTE (else NULL/zero):
          */
         char       *ctename;        /* name of the WITH list item */
         Index       ctelevelsup;    /* number of query levels up */
         bool        self_reference; /* is this a recursive self-reference? */
     
         /*
          * Fields valid for table functions, values, CTE and ENR RTEs (else NIL):
          *
          * We need these for CTE RTEs so that the types of self-referential
          * columns are well-defined.  For VALUES RTEs, storing these explicitly
          * saves having to re-determine the info by scanning the values_lists. For
          * ENRs, we store the types explicitly here (we could get the information
          * from the catalogs if 'relid' was supplied, but we'd still need these
          * for TupleDesc-based ENRs, so we might as well always store the type
          * info here).
          *
          * For ENRs only, we have to consider the possibility of dropped columns.
          * A dropped column is included in these lists, but it will have zeroes in
          * all three lists (as well as an empty-string entry in eref).  Testing
          * for zero coltype is the standard way to detect a dropped column.
          */
         List       *coltypes;       /* OID list of column type OIDs */
         List       *coltypmods;     /* integer list of column typmods */
         List       *colcollations;  /* OID list of column collation OIDs */
     
         /*
          * Fields valid for ENR RTEs (else NULL/zero):
          */
         char       *enrname;        /* name of ephemeral named relation */
         double      enrtuples;      /* estimated or actual from caller */
     
         /*
          * Fields valid in all RTEs:
          */
         Alias      *alias;          /* user-written alias clause, if any */
         Alias      *eref;           /* expanded reference names */
         bool        lateral;        /* subquery, function, or values is LATERAL? */
         bool        inh;            /* inheritance requested? */
         bool        inFromCl;       /* present in FROM clause? */
         AclMode     requiredPerms;  /* bitmask of required access permissions */
         Oid         checkAsUser;    /* if valid, check access as this role */
         Bitmapset  *selectedCols;   /* columns needing SELECT permission */
         Bitmapset  *insertedCols;   /* columns needing INSERT permission */
         Bitmapset  *updatedCols;    /* columns needing UPDATE permission */
         List       *securityQuals;  /* security barrier quals to apply, if any */
     } RangeTblEntry;
    
    

**FromExpr/JoinExpr**

    
    
     /*
      * RangeTblRef - reference to an entry in the query's rangetable
      *
      * We could use direct pointers to the RT entries and skip having these
      * nodes, but multiple pointers to the same node in a querytree cause
      * lots of headaches, so it seems better to store an index into the RT.
      */
     typedef struct RangeTblRef
     {
         NodeTag     type;
         int         rtindex;
     } RangeTblRef;
     
     /*----------      * JoinExpr - for SQL JOIN expressions
      *
      * isNatural, usingClause, and quals are interdependent.  The user can write
      * only one of NATURAL, USING(), or ON() (this is enforced by the grammar).
      * If he writes NATURAL then parse analysis generates the equivalent USING()
      * list, and from that fills in "quals" with the right equality comparisons.
      * If he writes USING() then "quals" is filled with equality comparisons.
      * If he writes ON() then only "quals" is set.  Note that NATURAL/USING
      * are not equivalent to ON() since they also affect the output column list.
      *
      * alias is an Alias node representing the AS alias-clause attached to the
      * join expression, or NULL if no clause.  NB: presence or absence of the
      * alias has a critical impact on semantics, because a join with an alias
      * restricts visibility of the tables/columns inside it.
      *
      * During parse analysis, an RTE is created for the Join, and its index
      * is filled into rtindex.  This RTE is present mainly so that Vars can
      * be created that refer to the outputs of the join.  The planner sometimes
      * generates JoinExprs internally; these can have rtindex = 0 if there are
      * no join alias variables referencing such joins.
      *----------      */
     typedef struct JoinExpr
     {
         NodeTag     type;
         JoinType    jointype;       /* type of join */
         bool        isNatural;      /* Natural join? Will need to shape table */
         Node       *larg;           /* left subtree */
         Node       *rarg;           /* right subtree */
         List       *usingClause;    /* USING clause, if any (list of String) */
         Node       *quals;          /* qualifiers on join, if any */
         Alias      *alias;          /* user-written alias clause, if any */
         int         rtindex;        /* RT index assigned for join, or 0 */
     } JoinExpr;
     
     /*----------      * FromExpr - represents a FROM ... WHERE ... construct
      *
      * This is both more flexible than a JoinExpr (it can have any number of
      * children, including zero) and less so --- we don't need to deal with
      * aliases and so on.  The output column set is implicitly just the union
      * of the outputs of the children.
      *----------      */
     typedef struct FromExpr
     {
         NodeTag     type;
         List       *fromlist;       /* List of join subtrees */
         Node       *quals;          /* qualifiers on join, if any */
     } FromExpr;
     
    

**TargetEntry**

    
    
     
     /*--------------------      * TargetEntry -      *     a target entry (used in query target lists)
      *
      * Strictly speaking, a TargetEntry isn't an expression node (since it can't
      * be evaluated by ExecEvalExpr).  But we treat it as one anyway, since in
      * very many places it's convenient to process a whole query targetlist as a
      * single expression tree.
      *
      * In a SELECT's targetlist, resno should always be equal to the item's
      * ordinal position (counting from 1).  However, in an INSERT or UPDATE
      * targetlist, resno represents the attribute number of the destination
      * column for the item; so there may be missing or out-of-order resnos.
      * It is even legal to have duplicated resnos; consider
      *      UPDATE table SET arraycol[1] = ..., arraycol[2] = ..., ...
      * The two meanings come together in the executor, because the planner
      * transforms INSERT/UPDATE tlists into a normalized form with exactly
      * one entry for each column of the destination table.  Before that's
      * happened, however, it is risky to assume that resno == position.
      * Generally get_tle_by_resno() should be used rather than list_nth()
      * to fetch tlist entries by resno, and only in SELECT should you assume
      * that resno is a unique identifier.
      *
      * resname is required to represent the correct column name in non-resjunk
      * entries of top-level SELECT targetlists, since it will be used as the
      * column title sent to the frontend.  In most other contexts it is only
      * a debugging aid, and may be wrong or even NULL.  (In particular, it may
      * be wrong in a tlist from a stored rule, if the referenced column has been
      * renamed by ALTER TABLE since the rule was made.  Also, the planner tends
      * to store NULL rather than look up a valid name for tlist entries in
      * non-toplevel plan nodes.)  In resjunk entries, resname should be either
      * a specific system-generated name (such as "ctid") or NULL; anything else
      * risks confusing ExecGetJunkAttribute!
      *
      * ressortgroupref is used in the representation of ORDER BY, GROUP BY, and
      * DISTINCT items.  Targetlist entries with ressortgroupref=0 are not
      * sort/group items.  If ressortgroupref>0, then this item is an ORDER BY,
      * GROUP BY, and/or DISTINCT target value.  No two entries in a targetlist
      * may have the same nonzero ressortgroupref --- but there is no particular
      * meaning to the nonzero values, except as tags.  (For example, one must
      * not assume that lower ressortgroupref means a more significant sort key.)
      * The order of the associated SortGroupClause lists determine the semantics.
      *
      * resorigtbl/resorigcol identify the source of the column, if it is a
      * simple reference to a column of a base table (or view).  If it is not
      * a simple reference, these fields are zeroes.
      *
      * If resjunk is true then the column is a working column (such as a sort key)
      * that should be removed from the final output of the query.  Resjunk columns
      * must have resnos that cannot duplicate any regular column's resno.  Also
      * note that there are places that assume resjunk columns come after non-junk
      * columns.
      *--------------------      */
     typedef struct TargetEntry
     {
         Expr        xpr;
         Expr       *expr;           /* expression to evaluate */
         AttrNumber  resno;          /* attribute number (see notes above) */
         char       *resname;        /* name of the column (could be NULL) */
         Index       ressortgroupref;    /* nonzero if referenced by a sort/group
                                          * clause */
         Oid         resorigtbl;     /* OID of column's source table */
         AttrNumber  resorigcol;     /* column's number in source table */
         bool        resjunk;        /* set to true to eliminate the attribute from
                                      * final target list */
     } TargetEntry;
    
    

**OpExpr**

    
    
    /*
      * OpExpr - expression node for an operator invocation
      *
      * Semantically, this is essentially the same as a function call.
      *
      * Note that opfuncid is not necessarily filled in immediately on creation
      * of the node.  The planner makes sure it is valid before passing the node
      * tree to the executor, but during parsing/planning opfuncid can be 0.
      */
     typedef struct OpExpr
     {
         Expr        xpr;
         Oid         opno;           /* PG_OPERATOR OID of the operator */
         Oid         opfuncid;       /* PG_PROC OID of underlying function */
         Oid         opresulttype;   /* PG_TYPE OID of result value */
         bool        opretset;       /* true if operator returns set */
         Oid         opcollid;       /* OID of collation of result */
         Oid         inputcollid;    /* OID of collation that operator should use */
         List       *args;           /* arguments to the operator (1 or 2) */
         int         location;       /* token location, or -1 if unknown */
     } OpExpr;
    
    

### 三、小结

1、Query树结构：一棵树关键的信息包括rtable（相关的数据表或子查询等）、jointree（连接树信息）、targetList（输出的目标字段等信息）等；  
2、数据结构：重要的数据结构包括RangeTblEntry、FromExpr、JoinExpr等。

