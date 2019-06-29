本节简单介绍了PG查询优化对表达式预处理中连接Var(RTE中的Var,其中RTE_KIND=RTE_JOIN)溯源的过程。处理逻辑在主函数subquery_planner中通过调用flatten_join_alias_vars函数实现，该函数位于src/backend/optimizer/util/var.c文件中。

### 一、基本概念

连接Var溯源，意思是把连接产生的中间结果（中间结果也是Relation关系的一种）的投影替换为实际存在的关系的列（在PG中通过Var表示）。  
如下面的SQL语句：

    
    
    select a.*,b.grbh,b.je 
    from t_dwxx a,
         lateral (select t1.dwbh,t1.grbh,t2.je 
                  from t_grxx t1 inner join t_jfxx t2 
                       on t1.dwbh = a.dwbh 
                          and t1.grbh = t2.grbh) b;
    
    

b与a连接运算,查询树Query中的投影b.grbh和b.je这两列如需依赖关系b(子查询产生的中间结果),则需要把中间关系的投影列替换为实际的Relation的投影列,即t_grxx和t_jfxx的数据列.

PG源码中的注释：

    
    
         /*
          * If the query has any join RTEs, replace join alias variables with
          * base-relation variables.  We must do this first, since any expressions
          * we may extract from the joinaliasvars lists have not been preprocessed.
          * For example, if we did this after sublink processing, sublinks expanded
          * out from join aliases would not get processed.  But we can skip this in
          * non-lateral RTE functions, VALUES lists, and TABLESAMPLE clauses, since
          * they can't contain any Vars of the current query level.
          */
         if (root->hasJoinRTEs &&
             !(kind == EXPRKIND_RTFUNC ||
               kind == EXPRKIND_VALUES ||
               kind == EXPRKIND_TABLESAMPLE ||
               kind == EXPRKIND_TABLEFUNC))
            expr = flatten_join_alias_vars(root, expr);
    

### 二、源码解读

**flatten_join_alias_vars**

    
    
    /*
      * flatten_join_alias_vars
      *    Replace Vars that reference JOIN outputs with references to the original
      *    relation variables instead.  This allows quals involving such vars to be
      *    pushed down.  Whole-row Vars that reference JOIN relations are expanded
      *    into RowExpr constructs that name the individual output Vars.  This
      *    is necessary since we will not scan the JOIN as a base relation, which
      *    is the only way that the executor can directly handle whole-row Vars.
      *
      * This also adjusts relid sets found in some expression node types to
      * substitute the contained base rels for any join relid.
      *
      * If a JOIN contains sub-selects that have been flattened, its join alias
      * entries might now be arbitrary expressions, not just Vars.  This affects
      * this function in one important way: we might find ourselves inserting
      * SubLink expressions into subqueries, and we must make sure that their
      * Query.hasSubLinks fields get set to true if so.  If there are any
      * SubLinks in the join alias lists, the outer Query should already have
      * hasSubLinks = true, so this is only relevant to un-flattened subqueries.
      *
      * NOTE: this is used on not-yet-planned expressions.  We do not expect it
      * to be applied directly to the whole Query, so if we see a Query to start
      * with, we do want to increment sublevels_up (this occurs for LATERAL
      * subqueries).
      */
     Node *
     flatten_join_alias_vars(PlannerInfo *root, Node *node)
     {
         flatten_join_alias_vars_context context;
     
         context.root = root;
         context.sublevels_up = 0;
         /* flag whether join aliases could possibly contain SubLinks */
         context.possible_sublink = root->parse->hasSubLinks;
         /* if hasSubLinks is already true, no need to work hard */
         context.inserted_sublink = root->parse->hasSubLinks;
         //调用flatten_join_alias_vars_mutator处理Vars
         return flatten_join_alias_vars_mutator(node, &context);
     }
     
     static Node *
     flatten_join_alias_vars_mutator(Node *node,
                                     flatten_join_alias_vars_context *context)
     {
         if (node == NULL)
             return NULL;
         if (IsA(node, Var))//Var类型
         {
             Var        *var = (Var *) node;
             RangeTblEntry *rte;
             Node       *newvar;
     
             /* No change unless Var belongs to a JOIN of the target level */
             if (var->varlevelsup != context->sublevels_up)
                 return node;        /* no need to copy, really */
             rte = rt_fetch(var->varno, context->root->parse->rtable);
             if (rte->rtekind != RTE_JOIN)
                 return node;
             //在rte->rtekind == RTE_JOIN时才需要处理
             if (var->varattno == InvalidAttrNumber)
             {
                 /* Must expand whole-row reference */
                 RowExpr    *rowexpr;
                 List       *fields = NIL;
                 List       *colnames = NIL;
                 AttrNumber  attnum;
                 ListCell   *lv;
                 ListCell   *ln;
     
                 attnum = 0;
                 Assert(list_length(rte->joinaliasvars) == list_length(rte->eref->colnames));
                 forboth(lv, rte->joinaliasvars, ln, rte->eref->colnames)
                 {
                     newvar = (Node *) lfirst(lv);
                     attnum++;
                     /* Ignore dropped columns */
                     if (newvar == NULL)
                         continue;
                     newvar = copyObject(newvar);
     
                     /*
                      * If we are expanding an alias carried down from an upper
                      * query, must adjust its varlevelsup fields.
                      */
                     if (context->sublevels_up != 0)
                         IncrementVarSublevelsUp(newvar, context->sublevels_up, 0);
                     /* Preserve original Var's location, if possible */
                     if (IsA(newvar, Var))
                         ((Var *) newvar)->location = var->location;
                     /* Recurse in case join input is itself a join */
                     /* (also takes care of setting inserted_sublink if needed) */
                     newvar = flatten_join_alias_vars_mutator(newvar, context);
                     fields = lappend(fields, newvar);
                     /* We need the names of non-dropped columns, too */
                     colnames = lappend(colnames, copyObject((Node *) lfirst(ln)));
                 }
                 rowexpr = makeNode(RowExpr);
                 rowexpr->args = fields;
                 rowexpr->row_typeid = var->vartype;
                 rowexpr->row_format = COERCE_IMPLICIT_CAST;
                 rowexpr->colnames = colnames;
                 rowexpr->location = var->location;
     
                 return (Node *) rowexpr;
             }
     
             /* Expand join alias reference */
             //扩展join alias Var
             Assert(var->varattno > 0);
             newvar = (Node *) list_nth(rte->joinaliasvars, var->varattno - 1);
             Assert(newvar != NULL);
             newvar = copyObject(newvar);
     
             /*
              * If we are expanding an alias carried down from an upper query, must
              * adjust its varlevelsup fields.
              */
             if (context->sublevels_up != 0)
                 IncrementVarSublevelsUp(newvar, context->sublevels_up, 0);
     
             /* Preserve original Var's location, if possible */
             if (IsA(newvar, Var))
                 ((Var *) newvar)->location = var->location;
     
             /* Recurse in case join input is itself a join */
             newvar = flatten_join_alias_vars_mutator(newvar, context);
     
             /* Detect if we are adding a sublink to query */
             if (context->possible_sublink && !context->inserted_sublink)
                 context->inserted_sublink = checkExprHasSubLink(newvar);
     
             return newvar;
         }
         if (IsA(node, PlaceHolderVar))//占位符
         {
             /* Copy the PlaceHolderVar node with correct mutation of subnodes */
             PlaceHolderVar *phv;
     
             phv = (PlaceHolderVar *) expression_tree_mutator(node,
                                                              flatten_join_alias_vars_mutator,
                                                              (void *) context);
             /* now fix PlaceHolderVar's relid sets */
             if (phv->phlevelsup == context->sublevels_up)
             {
                 phv->phrels = alias_relid_set(context->root,
                                               phv->phrels);
             }
             return (Node *) phv;
         }
     
         if (IsA(node, Query))//查询树
         {
             /* Recurse into RTE subquery or not-yet-planned sublink subquery */
             Query      *newnode;
             bool        save_inserted_sublink;
     
             context->sublevels_up++;
             save_inserted_sublink = context->inserted_sublink;
             context->inserted_sublink = ((Query *) node)->hasSubLinks;
             newnode = query_tree_mutator((Query *) node,
                                          flatten_join_alias_vars_mutator,
                                          (void *) context,
                                          QTW_IGNORE_JOINALIASES);
             newnode->hasSubLinks |= context->inserted_sublink;
             context->inserted_sublink = save_inserted_sublink;
             context->sublevels_up--;
             return (Node *) newnode;
         }
         /* Already-planned tree not supported */
         Assert(!IsA(node, SubPlan));
         /* Shouldn't need to handle these planner auxiliary nodes here */
         Assert(!IsA(node, SpecialJoinInfo));
         Assert(!IsA(node, PlaceHolderInfo));
         Assert(!IsA(node, MinMaxAggInfo));
         //其他表达式
         return expression_tree_mutator(node, flatten_join_alias_vars_mutator,
                                        (void *) context);
     }
     
    

**query_tree_mutator**

    
    
     /*
      * query_tree_mutator --- initiate modification of a Query's expressions
      *
      * This routine exists just to reduce the number of places that need to know
      * where all the expression subtrees of a Query are.  Note it can be used
      * for starting a walk at top level of a Query regardless of whether the
      * mutator intends to descend into subqueries.  It is also useful for
      * descending into subqueries within a mutator.
      *
      * Some callers want to suppress mutating of certain items in the Query,
      * typically because they need to process them specially, or don't actually
      * want to recurse into subqueries.  This is supported by the flags argument,
      * which is the bitwise OR of flag values to suppress mutating of
      * indicated items.  (More flag bits may be added as needed.)
      *
      * Normally the Query node itself is copied, but some callers want it to be
      * modified in-place; they must pass QTW_DONT_COPY_QUERY in flags.  All
      * modified substructure is safely copied in any case.
      */
     Query *
     query_tree_mutator(Query *query,
                        Node *(*mutator) (),
                        void *context,
                        int flags)//遍历查询树
     {
         Assert(query != NULL && IsA(query, Query));
     
         if (!(flags & QTW_DONT_COPY_QUERY))
         {
             Query      *newquery;
     
             FLATCOPY(newquery, query, Query);
             query = newquery;
         }
     
         MUTATE(query->targetList, query->targetList, List *);//投影列
         MUTATE(query->withCheckOptions, query->withCheckOptions, List *);
         MUTATE(query->onConflict, query->onConflict, OnConflictExpr *);
         MUTATE(query->returningList, query->returningList, List *);
         MUTATE(query->jointree, query->jointree, FromExpr *);
         MUTATE(query->setOperations, query->setOperations, Node *);
         MUTATE(query->havingQual, query->havingQual, Node *);
         MUTATE(query->limitOffset, query->limitOffset, Node *);
         MUTATE(query->limitCount, query->limitCount, Node *);
         if (!(flags & QTW_IGNORE_CTE_SUBQUERIES))
             MUTATE(query->cteList, query->cteList, List *);
         else                        /* else copy CTE list as-is */
             query->cteList = copyObject(query->cteList);
         query->rtable = range_table_mutator(query->rtable,
                                             mutator, context, flags);//RTE
         return query;
     }
    

**range_table_mutator**

    
    
     /*
      * range_table_mutator is just the part of query_tree_mutator that processes
      * a query's rangetable.  This is split out since it can be useful on
      * its own.
      */
     List *
     range_table_mutator(List *rtable,
                         Node *(*mutator) (),
                         void *context,
                         int flags)
     {
         List       *newrt = NIL;
         ListCell   *rt;
     
         foreach(rt, rtable)//遍历RTE
         {
             RangeTblEntry *rte = (RangeTblEntry *) lfirst(rt);
             RangeTblEntry *newrte;
     
             FLATCOPY(newrte, rte, RangeTblEntry);
             switch (rte->rtekind)
             {
                 case RTE_RELATION:
                     MUTATE(newrte->tablesample, rte->tablesample,
                            TableSampleClause *);
                     /* we don't bother to copy eref, aliases, etc; OK? */
                     break;
                 case RTE_CTE:
                 case RTE_NAMEDTUPLESTORE:
                     /* nothing to do */
                     break;
                 case RTE_SUBQUERY:
                     if (!(flags & QTW_IGNORE_RT_SUBQUERIES))
                     {
                         CHECKFLATCOPY(newrte->subquery, rte->subquery, Query);
                         MUTATE(newrte->subquery, newrte->subquery, Query *);//遍历处理子查询
                     }
                     else
                     {
                         /* else, copy RT subqueries as-is */
                         newrte->subquery = copyObject(rte->subquery);
                     }
                     break;
                 case RTE_JOIN://连接,遍历处理joinaliasvars
                     if (!(flags & QTW_IGNORE_JOINALIASES))
                         MUTATE(newrte->joinaliasvars, rte->joinaliasvars, List *);
                     else
                     {
                         /* else, copy join aliases as-is */
                         newrte->joinaliasvars = copyObject(rte->joinaliasvars);
                     }
                     break;
                 case RTE_FUNCTION:
                     MUTATE(newrte->functions, rte->functions, List *);
                     break;
                 case RTE_TABLEFUNC:
                     MUTATE(newrte->tablefunc, rte->tablefunc, TableFunc *);
                     break;
                 case RTE_VALUES:
                     MUTATE(newrte->values_lists, rte->values_lists, List *);
                     break;
             }
             MUTATE(newrte->securityQuals, rte->securityQuals, List *);
             newrt = lappend(newrt, newrte);
         }
         return newrt;
     }
    

### 三、跟踪分析

在PG11中,没有进入"Expand join alias reference"的实现逻辑,猜测在上拉子查询的时候已作优化.

### 四、小结

1、优化过程：介绍了通过遍历的方式实现joinaliasvars链表变量中的处理；  
2、遍历处理：通过统一的遍历方式，改变XX_mutator函数对Node进行处理。

