本文简单介绍了PG执行SQL的流程，重点介绍了查询语句的解析（Parse）过程。

### 一、SQL执行流程

PG执行SQL的过程有以下几个步骤：  
第一步，根据输入的SQL语句执行SQL Parse，进行词法和语法分析等，最终生成解析树；  
第二步，根据解析树，执行查询逻辑/物理优化、查询重写，最终生成查询树；  
第三步，根据查询树，生成执行计划；  
第四步，执行器根据执行计划，执行SQL。

### 二、SQL解析

如前所述，PG的SQL Parse（解析）过程由函数pg_parse_query实现，在exec_simple_query函数中调用。  
代码如下：

    
    
     /*
      * Do raw parsing (only).
      *
      * A list of parsetrees (RawStmt nodes) is returned, since there might be
      * multiple commands in the given string.
      *
      * NOTE: for interactive queries, it is important to keep this routine
      * separate from the analysis & rewrite stages.  Analysis and rewriting
      * cannot be done in an aborted transaction, since they require access to
      * database tables.  So, we rely on the raw parser to determine whether
      * we've seen a COMMIT or ABORT command; when we are in abort state, other
      * commands are not processed any further than the raw parse stage.
      */
     List *
     pg_parse_query(const char *query_string)
     {
         List       *raw_parsetree_list;
     
         TRACE_POSTGRESQL_QUERY_PARSE_START(query_string);
     
         if (log_parser_stats)
             ResetUsage();
     
         raw_parsetree_list = raw_parser(query_string);
     
         if (log_parser_stats)
             ShowUsage("PARSER STATISTICS");
     
     #ifdef COPY_PARSE_PLAN_TREES
         /* Optional debugging check: pass raw parsetrees through copyObject() */
         {
             List       *new_list = copyObject(raw_parsetree_list);
     
             /* This checks both copyObject() and the equal() routines... */
             if (!equal(new_list, raw_parsetree_list))
                 elog(WARNING, "copyObject() failed to produce an equal raw parse tree");
             else
                 raw_parsetree_list = new_list;
         }
     #endif
     
         TRACE_POSTGRESQL_QUERY_PARSE_DONE(query_string);
     
         return raw_parsetree_list;
     }
     
     /*
      * raw_parser
      *      Given a query in string form, do lexical and grammatical analysis.
      *
      * Returns a list of raw (un-analyzed) parse trees.  The immediate elements
      * of the list are always RawStmt nodes.
      */
     List *
     raw_parser(const char *str)
     {
         core_yyscan_t yyscanner;
         base_yy_extra_type yyextra;
         int         yyresult;
     
         /* initialize the flex scanner */
         yyscanner = scanner_init(str, &yyextra.core_yy_extra,
                                  ScanKeywords, NumScanKeywords);
     
         /* base_yylex() only needs this much initialization */
         yyextra.have_lookahead = false;
     
         /* initialize the bison parser */
         parser_init(&yyextra);
     
         /* Parse! */
         yyresult = base_yyparse(yyscanner);
     
         /* Clean up (release memory) */
         scanner_finish(yyscanner);
     
         if (yyresult)               /* error */
             return NIL;
     
         return yyextra.parsetree;
     }
     
    

_重要的数据结构:SelectStmt结构体_

    
    
    /* ----------------------      *      Select Statement
      *
      * A "simple" SELECT is represented in the output of gram.y by a single
      * SelectStmt node; so is a VALUES construct.  A query containing set
      * operators (UNION, INTERSECT, EXCEPT) is represented by a tree of SelectStmt
      * nodes, in which the leaf nodes are component SELECTs and the internal nodes
      * represent UNION, INTERSECT, or EXCEPT operators.  Using the same node
      * type for both leaf and internal nodes allows gram.y to stick ORDER BY,
      * LIMIT, etc, clause values into a SELECT statement without worrying
      * whether it is a simple or compound SELECT.
      * ----------------------      */
     typedef enum SetOperation
     {
         SETOP_NONE = 0,
         SETOP_UNION,
         SETOP_INTERSECT,
         SETOP_EXCEPT
     } SetOperation;
     
     typedef struct SelectStmt
     {
         NodeTag     type;
     
         /*
          * These fields are used only in "leaf" SelectStmts.
          */
         List       *distinctClause; /* NULL, list of DISTINCT ON exprs, or
                                      * lcons(NIL,NIL) for all (SELECT DISTINCT) */
         IntoClause *intoClause;     /* target for SELECT INTO */
         List       *targetList;     /* the target list (of ResTarget) */
         List       *fromClause;     /* the FROM clause */
         Node       *whereClause;    /* WHERE qualification */
         List       *groupClause;    /* GROUP BY clauses */
         Node       *havingClause;   /* HAVING conditional-expression */
         List       *windowClause;   /* WINDOW window_name AS (...), ... */
     
         /*
          * In a "leaf" node representing a VALUES list, the above fields are all
          * null, and instead this field is set.  Note that the elements of the
          * sublists are just expressions, without ResTarget decoration. Also note
          * that a list element can be DEFAULT (represented as a SetToDefault
          * node), regardless of the context of the VALUES list. It's up to parse
          * analysis to reject that where not valid.
          */
         List       *valuesLists;    /* untransformed list of expression lists */
     
         /*
          * These fields are used in both "leaf" SelectStmts and upper-level
          * SelectStmts.
          */
         List       *sortClause;     /* sort clause (a list of SortBy's) */
         Node       *limitOffset;    /* # of result tuples to skip */
         Node       *limitCount;     /* # of result tuples to return */
         List       *lockingClause;  /* FOR UPDATE (list of LockingClause's) */
         WithClause *withClause;     /* WITH clause */
     
         /*
          * These fields are used only in upper-level SelectStmts.
          */
         SetOperation op;            /* type of set op */
         bool        all;            /* ALL specified? */
         struct SelectStmt *larg;    /* left child */
         struct SelectStmt *rarg;    /* right child */
         /* Eventually add fields for CORRESPONDING spec here */
     } SelectStmt;
    
    

_重要的结构体:Value_

    
    
     /*----------------------      *      Value node
      *
      * The same Value struct is used for five node types: T_Integer,
      * T_Float, T_String, T_BitString, T_Null.
      *
      * Integral values are actually represented by a machine integer,
      * but both floats and strings are represented as strings.
      * Using T_Float as the node type simply indicates that
      * the contents of the string look like a valid numeric literal.
      *
      * (Before Postgres 7.0, we used a double to represent T_Float,
      * but that creates loss-of-precision problems when the value is
      * ultimately destined to be converted to NUMERIC.  Since Value nodes
      * are only used in the parsing process, not for runtime data, it's
      * better to use the more general representation.)
      *
      * Note that an integer-looking string will get lexed as T_Float if
      * the value is too large to fit in an 'int'.
      *
      * Nulls, of course, don't need the value part at all.
      *----------------------      */
     typedef struct Value
     {
         NodeTag     type;           /* tag appropriately (eg. T_String) */
         union ValUnion
         {
             int         ival;       /* machine integer */
             char       *str;        /* string */
         }           val;
     } Value;
     
     #define intVal(v)       (((Value *)(v))->val.ival)
     #define floatVal(v)     atof(((Value *)(v))->val.str)
     #define strVal(v)       (((Value *)(v))->val.str)
     
    

实现过程本节暂时搁置，先看过程执行的结果，函数pg_parse_query返回的结果是链表List，其中的元素是RawStmt，具体的结构需根据NodeTag确定（
**这样的做法类似于Java/C++的多态** ）。  
_测试数据_

    
    
    testdb=# -- 单位信息
    testdb=# drop table if exists t_dwxx;
    ues('Y有限公司','1002','北京市海淀区');
    insert into t_dwxx(dwmc,dwbh,dwdz) values('Z有限公司','1003','广西南宁市五象区');
    NOTICE:  table "t_dwxx" does not exist, skipping
    DROP TABLE
    testdb=# create table t_dwxx(dwmc varchar(100),dwbh varchar(10),dwdz varchar(100));
    CREATE TABLE
    testdb=# 
    testdb=# insert into t_dwxx(dwmc,dwbh,dwdz) values('X有限公司','1001','广东省广州市荔湾区');
    INSERT 0 1
    testdb=# insert into t_dwxx(dwmc,dwbh,dwdz) values('Y有限公司','1002','北京市海淀区');
    INSERT 0 1
    testdb=# insert into t_dwxx(dwmc,dwbh,dwdz) values('Z有限公司','1003','广西南宁市五象区');
    INSERT 0 1
    testdb=# -- 个人信息
    testdb=# drop table if exists t_grxx;
    NOTICE:  table "t_grxx" does not exist, skipping
    DROP TABLE
    testdb=# create table t_grxx(dwbh varchar(10),grbh varchar(10),xm varchar(20),nl int);
    CREATE TABLE
    insert into t_grxx(dwbh,grbh,xm,nl) values('1002','903','王五',43);
    testdb=# 
    testdb=# insert into t_grxx(dwbh,grbh,xm,nl) values('1001','901','张三',23);
    INSERT 0 1
    testdb=# insert into t_grxx(dwbh,grbh,xm,nl) values('1002','902','李四',33);
    INSERT 0 1
    testdb=# insert into t_grxx(dwbh,grbh,xm,nl) values('1002','903','王五',43);
    INSERT 0 1
    testdb=# -- 个人缴费信息
    testdb=# drop table if exists t_jfxx;
    NOTICE:  table "t_jfxx" does not exist, skipping
    DROP TABLE
    testdb=# create table t_jfxx(grbh varchar(10),ny varchar(10),je float);
    CREATE TABLE
    testdb=# 
    testdb=# insert into t_jfxx(grbh,ny,je) values('901','201801',401.30);
    insert into t_jfxx(grbh,ny,je) values('901','201802',401.30);
    insert into t_jfxx(grbh,ny,je) values('901','201803',401.30);
    insert into t_jfxx(grbh,ny,je) values('902','201801',513.30);
    insert into t_jfxx(grbh,ny,je) values('902','201802',513.30);
    insert into t_jfxx(grbh,ny,je) values('902','201804',513.30);
    insert into t_jfxx(grbh,ny,je) values('903','201801',372.22);
    insert into t_jfxx(grbh,ny,je) values('903','201804',372.22);
    testdb=# insert into t_jfxx(grbh,ny,je) values('901','201801',401.30);
    INSERT 0 1
    testdb=# insert into t_jfxx(grbh,ny,je) values('901','201802',401.30);
    INSERT 0 1
    testdb=# insert into t_jfxx(grbh,ny,je) values('901','201803',401.30);
    INSERT 0 1
    testdb=# insert into t_jfxx(grbh,ny,je) values('902','201801',513.10);
    INSERT 0 1
    testdb=# insert into t_jfxx(grbh,ny,je) values('902','201802',513.30);
    INSERT 0 1
    testdb=# insert into t_jfxx(grbh,ny,je) values('902','201804',513.30);
    INSERT 0 1
    testdb=# insert into t_jfxx(grbh,ny,je) values('903','201801',372.22);
    INSERT 0 1
    testdb=# insert into t_jfxx(grbh,ny,je) values('903','201804',372.22);
    INSERT 0 1
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               1560
    (1 row)
    -- 用于测试的查询语句
    testdb=# select t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je
    testdb-# from t_dwxx,t_grxx,t_jfxx
    testdb-# where t_dwxx.dwbh = t_grxx.dwbh 
    testdb-# and t_grxx.grbh = t_jfxx.grbh
    testdb-# and t_dwxx.dwbh IN ('1001','1002')
    testdb-# order by t_grxx.grbh
    testdb-# limit 8;
       dwmc    | grbh |  xm  |   ny   |   je   
    -----------+------+------+--------+--------     X有限公司 | 901  | 张三 | 201801 |  401.3
     X有限公司 | 901  | 张三 | 201802 |  401.3
     X有限公司 | 901  | 张三 | 201803 |  401.3
     Y有限公司 | 902  | 李四 | 201801 |  513.1
     Y有限公司 | 902  | 李四 | 201802 |  513.3
     Y有限公司 | 902  | 李四 | 201804 |  513.3
     Y有限公司 | 903  | 王五 | 201801 | 372.22
     Y有限公司 | 903  | 王五 | 201804 | 372.22
    (8 rows)
    

_结果分析_

    
    
    [xdb@localhost ~]$ gdb -p 1560
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b pg_parse_query
    Breakpoint 1 at 0x84c6c9: file postgres.c, line 615.
    (gdb) c
    Continuing.
    
    Breakpoint 1, pg_parse_query (
        query_string=0x1a46ef0 "select t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je\nfrom t_dwxx inner join t_grxx on t_dwxx.dwbh = t_grxx.dwbh\ninner join t_jfxx on t_grxx.grbh = t_jfxx.grbh\nwhere t_dwxx.dwbh IN ('1001','100"...) at postgres.c:615
    615     if (log_parser_stats)
    (gdb) n
    618     raw_parsetree_list = raw_parser(query_string);
    (gdb) 
    620     if (log_parser_stats)
    (gdb) 
    638     return raw_parsetree_list;
    (gdb) p *(RawStmt *)(raw_parsetree_list->head.data->ptr_value)
    $7 = {type = T_RawStmt, stmt = 0x1a48c00, stmt_location = 0, stmt_len = 232}
    (gdb) p *((RawStmt *)(raw_parsetree_list->head.data->ptr_value))->stmt
    $8 = {type = T_SelectStmt}
    #转换为实际类型SelectStmt 
    (gdb)  p *(SelectStmt *)((RawStmt *)(raw_parsetree_list->head.data->ptr_value))->stmt
    $16 = {type = T_SelectStmt, distinctClause = 0x0, intoClause = 0x0, targetList = 0x1a47b18, 
      fromClause = 0x1a48900, whereClause = 0x1a48b40, groupClause = 0x0, havingClause = 0x0, windowClause = 0x0, 
      valuesLists = 0x0, sortClause = 0x1afd858, limitOffset = 0x0, limitCount = 0x1afd888, lockingClause = 0x0, 
      withClause = 0x0, op = SETOP_NONE, all = false, larg = 0x0, rarg = 0x0}
    #设置临时变量
    (gdb) set $stmt=(SelectStmt *)((RawStmt *)(raw_parsetree_list->head.data->ptr_value))->stmt
    #查看结构体中的各个变量
    #------------------->targetList 
    (gdb) p *($stmt->targetList)
    $28 = {type = T_List, length = 5, head = 0x1a47af8, tail = 0x1a48128}
    #targetList有5个元素,分别对应t_dwxx.dwmc,t_grxx.grbh,t_grxx.xm,t_jfxx.ny,t_jfxx.je
    #先看第1个元素
    (gdb) set $restarget=(ResTarget *)($stmt->targetList->head.data->ptr_value)
    (gdb) p *$restarget->val
    $25 = {type = T_ColumnRef}
    (gdb) p *(ColumnRef *)$restarget->val
    $26 = {type = T_ColumnRef, fields = 0x1a47a08, location = 7}
    (gdb) p *((ColumnRef *)$restarget->val)->fields
    $27 = {type = T_List, length = 2, head = 0x1a47a88, tail = 0x1a479e8}
    (gdb) p *(Node *)(((ColumnRef *)$restarget->val)->fields)->head.data->ptr_value
    $32 = {type = T_String}
    #fields链表的第1个元素是数据表,第2个元素是数据列
    (gdb) p *(Value *)(((ColumnRef *)$restarget->val)->fields)->head.data->ptr_value
    $37 = {type = T_String, val = {ival = 27556248, str = 0x1a47998 "t_dwxx"}}
    (gdb) p *(Value *)(((ColumnRef *)$restarget->val)->fields)->tail.data->ptr_value
    $38 = {type = T_String, val = {ival = 27556272, str = 0x1a479b0 "dwmc"}}
    #其他类似
    
    #------------------->fromClause 
    (gdb) p *(Node *)($stmt->fromClause->head.data->ptr_value)
    $41 = {type = T_JoinExpr}
    (gdb) set $fromclause=(JoinExpr *)($stmt->fromClause->head.data->ptr_value)
    (gdb) p *$fromclause
    $42 = {type = T_JoinExpr, jointype = JOIN_INNER, isNatural = false, larg = 0x1a484f8, rarg = 0x1a48560, 
      usingClause = 0x0, quals = 0x1a487d0, alias = 0x0, rtindex = 0}
    
    #------------------->whereClause 
    (gdb)  p *(Node *)($stmt->whereClause)
    $44 = {type = T_A_Expr}
    (gdb)  p *(FromExpr *)($stmt->whereClause)
    $46 = {type = T_A_Expr, fromlist = 0x1a48bd0, quals = 0x1a489d0}
    
    #------------------->sortClause 
    (gdb)  p *(Node *)($stmt->sortClause->head.data->ptr_value)
    $48 = {type = T_SortBy}
    (gdb)  p *(SortBy *)($stmt->sortClause->head.data->ptr_value)
    $49 = {type = T_SortBy, node = 0x1a48db0, sortby_dir = SORTBY_DEFAULT, sortby_nulls = SORTBY_NULLS_DEFAULT, 
      useOp = 0x0, location = -1}
    
    #------------------->limitCount 
    (gdb)  p *(Node *)($stmt->limitCount)
    $50 = {type = T_A_Const}
    (gdb)  p *(Const *)($stmt->limitCount)
    $51 = {xpr = {type = T_A_Const}, consttype = 0, consttypmod = 216, constcollid = 0, constlen = 8, 
      constvalue = 231, constisnull = 16, constbyval = false, location = 0}
    
    

以上为简单的数据结构介绍，下一节将详细解析parseTree的Tree结构。

### 三、小结

1、SQL执行流程：简单介绍了SQL的执行流程，简单分为解析、查询优化、执行计划生成和执行这四步；  
2、SQL解析：解析执行后，结果存储在解析树中，分为distinctClause、intoClause、targetList等多个部分。

