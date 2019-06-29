本文简单介绍了PG根据查询树生成的执行计划的详细结构，生成执行计划的输入是上一节介绍的查询树Query。

### 一、PlannedStmt结构

生成执行计划在函数pg_plan_queries中实现,返回的是链表plantree_list,链表中的元素是PlannedStmt.  
PlannedStmt结构:

    
    
     /* ----------------      *      PlannedStmt node
      *
      * The output of the planner is a Plan tree headed by a PlannedStmt node.
      * PlannedStmt holds the "one time" information needed by the executor.
      *
      * For simplicity in APIs, we also wrap utility statements in PlannedStmt
      * nodes; in such cases, commandType == CMD_UTILITY, the statement itself
      * is in the utilityStmt field, and the rest of the struct is mostly dummy.
      * (We do use canSetTag, stmt_location, stmt_len, and possibly queryId.)
      * ----------------      */
     typedef struct PlannedStmt
     {
         NodeTag     type;
     
         CmdType     commandType;    /* select|insert|update|delete|utility */
     
         uint64      queryId;        /* query identifier (copied from Query) */
     
         bool        hasReturning;   /* is it insert|update|delete RETURNING? */
     
         bool        hasModifyingCTE;    /* has insert|update|delete in WITH? */
     
         bool        canSetTag;      /* do I set the command result tag? */
     
         bool        transientPlan;  /* redo plan when TransactionXmin changes? */
     
         bool        dependsOnRole;  /* is plan specific to current role? */
     
         bool        parallelModeNeeded; /* parallel mode required to execute? */
     
         int         jitFlags;       /* which forms of JIT should be performed */
     
         struct Plan *planTree;      /* tree of Plan nodes */
     
         List       *rtable;         /* list of RangeTblEntry nodes */
     
         /* rtable indexes of target relations for INSERT/UPDATE/DELETE */
         List       *resultRelations;    /* integer list of RT indexes, or NIL */
     
         /*
          * rtable indexes of non-leaf target relations for UPDATE/DELETE on all
          * the partitioned tables mentioned in the query.
          */
         List       *nonleafResultRelations;
     
         /*
          * rtable indexes of root target relations for UPDATE/DELETE; this list
          * maintains a subset of the RT indexes in nonleafResultRelations,
          * indicating the roots of the respective partition hierarchies.
          */
         List       *rootResultRelations;
     
         List       *subplans;       /* Plan trees for SubPlan expressions; note
                                      * that some could be NULL */
     
         Bitmapset  *rewindPlanIDs;  /* indices of subplans that require REWIND */
     
         List       *rowMarks;       /* a list of PlanRowMark's */
     
         List       *relationOids;   /* OIDs of relations the plan depends on */
     
         List       *invalItems;     /* other dependencies, as PlanInvalItems */
     
         List       *paramExecTypes; /* type OIDs for PARAM_EXEC Params */
     
         Node       *utilityStmt;    /* non-null if this is utility stmt */
     
         /* statement location in source string (copied from Query) */
         int         stmt_location;  /* start location, or -1 if unknown */
         int         stmt_len;       /* length in bytes; 0 means "rest of string" */
     } PlannedStmt;
     
     /* macro for fetching the Plan associated with a SubPlan node */
     #define exec_subplan_get_plan(plannedstmt, subplan) \
         ((Plan *) list_nth((plannedstmt)->subplans, (subplan)->plan_id - 1))
    

SQL语句:

    
    
    select t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je
    from t_dwxx,t_grxx,t_jfxx
    where t_dwxx.dwbh = t_grxx.dwbh 
      and t_grxx.grbh = t_jfxx.grbh
      and t_dwxx.dwbh IN ('1001','1002')
    order by t_grxx.grbh
    limit 8;
    
    select * from (
    select t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je
    from t_dwxx inner join t_grxx on t_dwxx.dwbh = t_grxx.dwbh
      inner join t_jfxx on t_grxx.grbh = t_jfxx.grbh
    where t_dwxx.dwbh IN ('1001')
    union all
    select t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je
    from t_dwxx inner join t_grxx on t_dwxx.dwbh = t_grxx.dwbh
      inner join t_jfxx on t_grxx.grbh = t_jfxx.grbh
    where t_dwxx.dwbh IN ('1002') 
    ) as ret
    order by ret.grbh
    limit 4;
    
    

跟踪分析:

    
    
    (gdb) b exec_simple_query
    Breakpoint 1 at 0x84cad8: file postgres.c, line 893.
    (gdb) c
    Continuing.
    
    Breakpoint 1, exec_simple_query (
        query_string=0x12b3ef0 "select * from (\nselect t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je\nfrom t_dwxx inner join t_grxx on t_dwxx.dwbh = t_grxx.dwbh\ninner join t_jfxx on t_grxx.grbh = t_jfxx.grbh\nwhere t_dwxx.dwbh"...) at postgres.c:893
    893   CommandDest dest = whereToSendOutput;
    1050      plantree_list = pg_plan_queries(querytree_list,
    (gdb) 
    1054      if (snapshot_set)
    (gdb) p (PlannedStmt *)plantree_list->head->data.ptr_value
    $3 = (PlannedStmt *) 0x7f1a23ea74c8
    (gdb) p *(PlannedStmt *)plantree_list->head->data.ptr_value
    $4 = {type = T_PlannedStmt, commandType = CMD_SELECT, queryId = 0, hasReturning = false, 
      hasModifyingCTE = false, canSetTag = true, transientPlan = false, dependsOnRole = false, 
      parallelModeNeeded = false, jitFlags = 0, planTree = 0x7f1a23ea27f8, rtable = 0x7f1a23ea29b8, 
      resultRelations = 0x0, nonleafResultRelations = 0x0, rootResultRelations = 0x0, subplans = 0x0, 
      rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x7f1a23ea3968, invalItems = 0x0, paramExecTypes = 0x0, 
      utilityStmt = 0x0, stmt_location = 0, stmt_len = 455}
    (gdb) set $pstmt=(PlannedStmt *)plantree_list->head->data.ptr_value
    (gdb) p *$pstmt->rtable
    $5 = {type = T_List, length = 13, head = 0x7f1a23ea2998, tail = 0x7f1a23ea5cb8}
    (gdb) p *(Node *)$pstmt->rtable->head->data.ptr_value
    $6 = {type = T_RangeTblEntry}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->data.ptr_value
    $7 = {type = T_RangeTblEntry, rtekind = RTE_SUBQUERY, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x12d8d40, eref = 0x13abb08, lateral = false, inh = true, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *((RangeTblEntry *)$pstmt->rtable->head->data.ptr_value)->alias
    $8 = {type = T_Alias, aliasname = 0x12d8d28 "ret", colnames = 0x0}
    (gdb) p *((RangeTblEntry *)$pstmt->rtable->head->data.ptr_value)->eref
    $9 = {type = T_Alias, aliasname = 0x13abb38 "ret", colnames = 0x13abba8}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->next->data.ptr_value
    $10 = {type = T_RangeTblEntry, rtekind = RTE_SUBQUERY, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x137ee90, eref = 0x137eee0, lateral = false, inh = false, inFromCl = false, 
      requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *((RangeTblEntry *)$pstmt->rtable->head->next->data.ptr_value)->alias
    $11 = {type = T_Alias, aliasname = 0x137eec0 "*SELECT* 1", colnames = 0x0}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->next->next->data.ptr_value
    $12 = {type = T_RangeTblEntry, rtekind = RTE_SUBQUERY, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x13818d8, eref = 0x1381928, lateral = false, inh = false, inFromCl = false, 
      requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *((RangeTblEntry *)$pstmt->rtable->head->next->next->data.ptr_value)->alias
    $13 = {type = T_Alias, aliasname = 0x1381908 "*SELECT* 2", colnames = 0x0}
    $14 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26754, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x1384418, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x1384598, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->data.ptr_value
    $15 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26757, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x13846e0, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x13848b8, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->next->data.ptr_value
    $16 = {type = T_RangeTblEntry, rtekind = RTE_JOIN, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x1384d40, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *((RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->next->data.ptr_value)->eref
    $17 = {type = T_Alias, aliasname = 0x1384d70 "unnamed_join", colnames = 0x1384d90}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->next->next->data.ptr_value
    $18 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26760, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x1385158, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x13852d8, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *((RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->next->next->data.ptr_value)->eref
    $19 = {type = T_Alias, aliasname = 0x1385188 "t_jfxx", colnames = 0x13851a0}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->next->next->next->data.ptr_value
    $20 = {type = T_RangeTblEntry, rtekind = RTE_JOIN, relid = 0, relkind = 0 '\000', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x13858b0, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 0, checkAsUser = 0, selectedCols = 0x0, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->next->next->next->next->data.ptr_value
    $21 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26754, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x7f1a23e95308, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x7f1a23e95458, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *((RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->next->next->next->next->data.ptr_value)->eref
    $22 = {type = T_Alias, aliasname = 0x138bf90 "t_dwxx", colnames = 0x7f1a23e95338}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->next->next->next->next->next->data.ptr_value
    $23 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26757, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x7f1a23e955a0, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x7f1a23e95778, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    (gdb) p *(RangeTblEntry *)$pstmt->rtable->head->next->next->next->next->next->next->next->next->next->data.ptr_value
    $24 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 26757, relkind = 114 'r', tablesample = 0x0, 
      subquery = 0x0, security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, 
      funcordinality = false, tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, 
      self_reference = false, coltypes = 0x0, coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, 
      enrtuples = 0, alias = 0x0, eref = 0x7f1a23e955a0, lateral = false, inh = false, inFromCl = true, 
      requiredPerms = 2, checkAsUser = 0, selectedCols = 0x7f1a23e95778, insertedCols = 0x0, updatedCols = 0x0, 
      securityQuals = 0x0}
    
    #--->planTree:Limit
    (gdb) p $pstmt->planTree
    $25 = (struct Plan *) 0x7f1a23ea27f8
    (gdb) p *$pstmt->planTree
    $26 = {type = T_Limit, startup_cost = 96.799999999999983, total_cost = 96.809999999999988, plan_rows = 4, 
      plan_width = 360, parallel_aware = false, parallel_safe = true, plan_node_id = 0, 
      targetlist = 0x7f1a23ea2d08, qual = 0x0, lefttree = 0x7f1a23ea2488, righttree = 0x0, initPlan = 0x0, 
      extParam = 0x0, allParam = 0x0}
    (gdb) p *$pstmt->relationOids
    $45 = {type = T_OidList, length = 6, head = 0x7f1a23ea3948, tail = 0x7f1a23ea5b88}
    (gdb) p *$pstmt->relationOids->head
    $46 = {data = {ptr_value = 0x6882, int_value = 26754, oid_value = 26754}, next = 0x7f1a23ea3ac8}
    
    #--->planTree->lefttree:Sort
    (gdb) p *$pstmt->planTree->lefttree
    $52 = {type = T_Sort, startup_cost = 96.799999999999983, total_cost = 96.83499999999998, plan_rows = 14, 
      plan_width = 360, parallel_aware = false, parallel_safe = true, plan_node_id = 1, 
      targetlist = 0x7f1a23ea30f8, qual = 0x0, lefttree = 0x7f1a23e9d078, righttree = 0x0, initPlan = 0x0, 
      extParam = 0x0, allParam = 0x0}
    (gdb) p *(Sort *)$pstmt->planTree->lefttree
    $53 = {plan = {type = T_Sort, startup_cost = 96.799999999999983, total_cost = 96.83499999999998, 
        plan_rows = 14, plan_width = 360, parallel_aware = false, parallel_safe = true, plan_node_id = 1, 
        targetlist = 0x7f1a23ea30f8, qual = 0x0, lefttree = 0x7f1a23e9d078, righttree = 0x0, initPlan = 0x0, 
        extParam = 0x0, allParam = 0x0}, numCols = 1, sortColIdx = 0x7f1a23e9dae8, 
      sortOperators = 0x7f1a23ea2440, collations = 0x7f1a23ea2458, nullsFirst = 0x7f1a23ea2470}
    (gdb) set $sort=(Sort *)$pstmt->planTree->lefttree
    (gdb) p *$sort->plan.lefttree
    $59 = {type = T_Append, startup_cost = 16.149999999999999, total_cost = 96.589999999999989, plan_rows = 14, 
      plan_width = 360, parallel_aware = false, parallel_safe = true, plan_node_id = 2, 
      targetlist = 0x7f1a23ea34e8, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, 
      allParam = 0x0}
    (gdb) p *(Append *)$sort->plan.lefttree
    $60 = {plan = {type = T_Append, startup_cost = 16.149999999999999, total_cost = 96.589999999999989, 
        plan_rows = 14, plan_width = 360, parallel_aware = false, parallel_safe = true, plan_node_id = 2, 
        targetlist = 0x7f1a23ea34e8, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, 
        allParam = 0x0}, appendplans = 0x7f1a23ea1030, first_partial_plan = 2, partitioned_rels = 0x0, 
      part_prune_infos = 0x0}
    
    ##--->planTree->lefttree->plan.lefttree:Append
    (gdb)set $append=(Append *)$sort->plan.lefttree
    (gdb) p *$append.appendplans
    $64 = {type = T_List, length = 2, head = 0x7f1a23ea1010, tail = 0x7f1a23ea2420}
    (gdb) p *(Node *)$append.appendplans->head->data.ptr_value
    $65 = {type = T_NestLoop}
    (gdb) p *(NestLoop *)$append.appendplans->head->data.ptr_value
    $73 = {join = {plan = {type = T_NestLoop, startup_cost = 16.149999999999999, total_cost = 48.189999999999998, 
          plan_rows = 7, plan_width = 360, parallel_aware = false, parallel_safe = true, plan_node_id = 4, 
          targetlist = 0x7f1a23ea3ff8, qual = 0x0, lefttree = 0x7f1a23e9f9a0, righttree = 0x7f1a23ea0e60, 
          initPlan = 0x0, extParam = 0x0, allParam = 0x0}, jointype = JOIN_INNER, inner_unique = false, 
        joinqual = 0x0}, nestParams = 0x0}
    (gdb) set $nl1=(NestLoop *)$append.appendplans->head->data.ptr_value
    (gdb) set $nl2=(NestLoop *)$append.appendplans->head->next->data.ptr_value
    (gdb) p *$nl1->join->plan->lefttree
    $79 = {type = T_SeqScan, startup_cost = 0, total_cost = 12, plan_rows = 1, plan_width = 256, 
      parallel_aware = false, parallel_safe = true, plan_node_id = 5, targetlist = 0x7f1a23ea4348, 
      qual = 0x7f1a23ea46c8, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) p *(SeqScan *)$nl1->join->plan->lefttree
    $80 = {plan = {type = T_SeqScan, startup_cost = 0, total_cost = 12, plan_rows = 1, plan_width = 256, 
        parallel_aware = false, parallel_safe = true, plan_node_id = 5, targetlist = 0x7f1a23ea4348, 
        qual = 0x7f1a23ea46c8, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}, 
      scanrelid = 4}
    (gdb) p *(SeqScan *)$nl1->join->plan->lefttree->qual
    $81 = {plan = {type = T_List, startup_cost = 6.9045796748682899e-310, total_cost = 6.9045796748682899e-310, 
        plan_rows = 0, plan_width = 64, parallel_aware = false, parallel_safe = false, plan_node_id = 19611104, 
        targetlist = 0x700000067, qual = 0x41300000001, lefttree = 0x640000000e, righttree = 0x700000000, 
        initPlan = 0xffffffff00000001, extParam = 0x0, allParam = 0x0}, scanrelid = 0}
    (gdb) p *(Node *)$nl1->join->plan->lefttree->qual
    $82 = {type = T_List}
    (gdb) p *(List *)$nl1->join->plan->lefttree->qual
    $83 = {type = T_List, length = 1, head = 0x7f1a23ea46a8, tail = 0x7f1a23ea46a8}
    (gdb) p *(Node *)$nl1->join->plan->lefttree->qual->head->data.ptr_value
    $84 = {type = T_OpExpr}
    (gdb) p *(OpExpr *)$nl1->join->plan->lefttree->qual->head->data.ptr_value
    $85 = {xpr = {type = T_OpExpr}, opno = 98, opfuncid = 67, opresulttype = 16, opretset = false, opcollid = 0, 
      inputcollid = 100, args = 0x7f1a23ea4608, location = -1}
    (gdb) set $opexpr=(OpExpr *)$nl1->join->plan->lefttree->qual->head->data.ptr_value
    (gdb) p *$opexpr->args
    $86 = {type = T_List, length = 2, head = 0x7f1a23ea45e8, tail = 0x7f1a23ea4688}
    (gdb) p *(Node *)$opexpr->args->head->data.ptr_value
    $87 = {type = T_RelabelType}
    (gdb) p *(RelabelType *)$opexpr->args->head->data.ptr_value
    $88 = {xpr = {type = T_RelabelType}, arg = 0x7f1a23ea4598, resulttype = 25, resulttypmod = -1, 
      resultcollid = 100, relabelformat = COERCE_IMPLICIT_CAST, location = -1}
    (gdb) p *((RelabelType *)$opexpr->args->head->data.ptr_value)->arg
    $89 = {type = T_Var}
    (gdb) p *(Var *)((RelabelType *)$opexpr->args->head->data.ptr_value)->arg
    $90 = {xpr = {type = T_Var}, varno = 4, varattno = 2, vartype = 1043, vartypmod = 14, varcollid = 100, 
      varlevelsup = 0, varnoold = 4, varoattno = 2, location = 110}
    (gdb) p *(RelabelType *)$opexpr->args->tail->data.ptr_value
    $91 = {xpr = {type = T_Const}, arg = 0x64ffffffff, resulttype = 4294967295, resulttypmod = 0, 
      resultcollid = 20488992, relabelformat = COERCE_EXPLICIT_CALL, location = 0}
    (gdb) p *(Const *)$opexpr->args->tail->data.ptr_value
    $92 = {xpr = {type = T_Const}, consttype = 25, consttypmod = -1, constcollid = 100, constlen = -1, 
      constvalue = 20488992, constisnull = false, constbyval = false, location = 205}
    #其他类似
    #DONE!
    

最终的计划树结构如下:

  

![](https://upload-images.jianshu.io/upload_images/8194836-69e162c2af21f0fe.png)

计划树

### 二、数据结构

**Plan**

    
    
     /* ----------------      *      Plan node
      *
      * All plan nodes "derive" from the Plan structure by having the
      * Plan structure as the first field.  This ensures that everything works
      * when nodes are cast to Plan's.  (node pointers are frequently cast to Plan*
      * when passed around generically in the executor)
      *
      * We never actually instantiate any Plan nodes; this is just the common
      * abstract superclass for all Plan-type nodes.
      * ----------------      */
     typedef struct Plan
     {
         NodeTag     type;
     
         /*
          * estimated execution costs for plan (see costsize.c for more info)
          */
         Cost        startup_cost;   /* cost expended before fetching any tuples */
         Cost        total_cost;     /* total cost (assuming all tuples fetched) */
     
         /*
          * planner's estimate of result size of this plan step
          */
         double      plan_rows;      /* number of rows plan is expected to emit */
         int         plan_width;     /* average row width in bytes */
     
         /*
          * information needed for parallel query
          */
         bool        parallel_aware; /* engage parallel-aware logic? */
         bool        parallel_safe;  /* OK to use as part of parallel plan? */
     
         /*
          * Common structural data for all Plan types.
          */
         int         plan_node_id;   /* unique across entire final plan tree */
         List       *targetlist;     /* target list to be computed at this node */
         List       *qual;           /* implicitly-ANDed qual conditions */
         struct Plan *lefttree;      /* input plan tree(s) */
         struct Plan *righttree;
         List       *initPlan;       /* Init Plan nodes (un-correlated expr
                                      * subselects) */
     
         /*
          * Information for management of parameter-change-driven rescanning
          *
          * extParam includes the paramIDs of all external PARAM_EXEC params
          * affecting this plan node or its children.  setParam params from the
          * node's initPlans are not included, but their extParams are.
          *
          * allParam includes all the extParam paramIDs, plus the IDs of local
          * params that affect the node (i.e., the setParams of its initplans).
          * These are _all_ the PARAM_EXEC params that affect this node.
          */
         Bitmapset  *extParam;
         Bitmapset  *allParam;
     } Plan;
    
    

**Limit**

    
    
     /* ----------------      *      limit node
      *
      * Note: as of Postgres 8.2, the offset and count expressions are expected
      * to yield int8, rather than int4 as before.
      * ----------------      */
     typedef struct Limit
     {
         Plan        plan;
         Node       *limitOffset;    /* OFFSET parameter, or NULL if none */
         Node       *limitCount;     /* COUNT parameter, or NULL if none */
     } Limit;
    

**Sort**

    
    
     /* ----------------      *      sort node
      * ----------------      */
     typedef struct Sort
     {
         Plan        plan;
         int         numCols;        /* number of sort-key columns */
         AttrNumber *sortColIdx;     /* their indexes in the target list */
         Oid        *sortOperators;  /* OIDs of operators to sort them by */
         Oid        *collations;     /* OIDs of collations */
         bool       *nullsFirst;     /* NULLS FIRST/LAST directions */
     } Sort;
    

**Append**

    
    
     /* ----------------      *   Append node -      *      Generate the concatenation of the results of sub-plans.
      * ----------------      */
     typedef struct Append
     {
         Plan        plan;
         List       *appendplans;
     
         /*
          * All 'appendplans' preceding this index are non-partial plans. All
          * 'appendplans' from this index onwards are partial plans.
          */
         int         first_partial_plan;
     
         /* RT indexes of non-leaf tables in a partition tree */
         List       *partitioned_rels;
     
         /* Info for run-time subplan pruning; NULL if we're not doing that */
         struct PartitionPruneInfo *part_prune_info;
     } Append;
    

**NestLoop**

    
    
     /* ----------------      *      nest loop join node
      *
      * The nestParams list identifies any executor Params that must be passed
      * into execution of the inner subplan carrying values from the current row
      * of the outer subplan.  Currently we restrict these values to be simple
      * Vars, but perhaps someday that'd be worth relaxing.  (Note: during plan
      * creation, the paramval can actually be a PlaceHolderVar expression; but it
      * must be a Var with varno OUTER_VAR by the time it gets to the executor.)
      * ----------------      */
     typedef struct NestLoop
     {
         Join        join;
         List       *nestParams;     /* list of NestLoopParam nodes */
     } NestLoop;
     
     typedef struct NestLoopParam
     {
         NodeTag     type;
         int         paramno;        /* number of the PARAM_EXEC Param to set */
         Var        *paramval;       /* outer-relation Var to assign to Param */
     } NestLoopParam;
     
     /*
      * ==========
      * Join nodes
      * ==========
      */
     /* ----------------      *      merge join node
      *
      * The expected ordering of each mergeable column is described by a btree
      * opfamily OID, a collation OID, a direction (BTLessStrategyNumber or
      * BTGreaterStrategyNumber) and a nulls-first flag.  Note that the two sides
      * of each mergeclause may be of different datatypes, but they are ordered the
      * same way according to the common opfamily and collation.  The operator in
      * each mergeclause must be an equality operator of the indicated opfamily.
      * ----------------      */
     typedef struct MergeJoin
     {
         Join        join;
         bool        skip_mark_restore;  /* Can we skip mark/restore calls? */
         List       *mergeclauses;   /* mergeclauses as expression trees */
         /* these are arrays, but have the same length as the mergeclauses list: */
         Oid        *mergeFamilies;  /* per-clause OIDs of btree opfamilies */
         Oid        *mergeCollations;    /* per-clause OIDs of collations */
         int        *mergeStrategies;    /* per-clause ordering (ASC or DESC) */
         bool       *mergeNullsFirst;    /* per-clause nulls ordering */
     } MergeJoin;
     
     /* ----------------      *      hash join node
      * ----------------      */
     typedef struct HashJoin
     {
         Join        join;
         List       *hashclauses;
     } HashJoin;
     
    
     /* ----------------      *      Join node
      *
      * jointype:    rule for joining tuples from left and right subtrees
      * inner_unique each outer tuple can match to no more than one inner tuple
      * joinqual:    qual conditions that came from JOIN/ON or JOIN/USING
      *              (plan.qual contains conditions that came from WHERE)
      *
      * When jointype is INNER, joinqual and plan.qual are semantically
      * interchangeable.  For OUTER jointypes, the two are *not* interchangeable;
      * only joinqual is used to determine whether a match has been found for
      * the purpose of deciding whether to generate null-extended tuples.
      * (But plan.qual is still applied before actually returning a tuple.)
      * For an outer join, only joinquals are allowed to be used as the merge
      * or hash condition of a merge or hash join.
      *
      * inner_unique is set if the joinquals are such that no more than one inner
      * tuple could match any given outer tuple.  This allows the executor to
      * skip searching for additional matches.  (This must be provable from just
      * the joinquals, ignoring plan.qual, due to where the executor tests it.)
      * ----------------      */
     typedef struct Join
     {
         Plan        plan;
         JoinType    jointype;
         bool        inner_unique;
         List       *joinqual;       /* JOIN quals (in addition to plan.qual) */
     } Join;
    

**SeqScan**

    
    
     /*
      * ==========
      * Scan nodes
      * ==========
      */
     typedef struct Scan
     {
         Plan        plan;
         Index       scanrelid;      /* relid is index into the range table */
     } Scan;
     
     /* ----------------      *      sequential scan node
      * ----------------      */
     typedef Scan SeqScan;
     
    

### 三、小结

1、PlannedStmt：这是已规划的SQL语句，可用于Executor执行；  
2、重要的数据结构：计划的基础结构Plan，“继承“的结构Limit、Sort等。

