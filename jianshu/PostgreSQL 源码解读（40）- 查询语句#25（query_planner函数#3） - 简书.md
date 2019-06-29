上一小节介绍了函数query_planner的主处理逻辑以及setup_simple_rel_arrays和setup_append_rel_array两个子函数的实现逻辑,本节继续介绍函数query_planner中的add_base_rels_to_query函数。

### 一、重要的数据结构

**Relation**

    
    
     /*
      * Here are the contents of a relation cache entry.
      */
     
     typedef struct RelationData
     {
         RelFileNode rd_node;        /* relation physical identifier */
         /* use "struct" here to avoid needing to include smgr.h: */
         struct SMgrRelationData *rd_smgr;   /* cached file handle, or NULL */
         int         rd_refcnt;      /* reference count */
         BackendId   rd_backend;     /* owning backend id, if temporary relation */
         bool        rd_islocaltemp; /* rel is a temp rel of this session */
         bool        rd_isnailed;    /* rel is nailed in cache */
         bool        rd_isvalid;     /* relcache entry is valid */
         char        rd_indexvalid;  /* state of rd_indexlist: 0 = not valid, 1 =
                                      * valid, 2 = temporarily forced */
         bool        rd_statvalid;   /* is rd_statlist valid? */
     
         /*
          * rd_createSubid is the ID of the highest subtransaction the rel has
          * survived into; or zero if the rel was not created in the current top
          * transaction.  This can be now be relied on, whereas previously it could
          * be "forgotten" in earlier releases. Likewise, rd_newRelfilenodeSubid is
          * the ID of the highest subtransaction the relfilenode change has
          * survived into, or zero if not changed in the current transaction (or we
          * have forgotten changing it). rd_newRelfilenodeSubid can be forgotten
          * when a relation has multiple new relfilenodes within a single
          * transaction, with one of them occurring in a subsequently aborted
          * subtransaction, e.g. BEGIN; TRUNCATE t; SAVEPOINT save; TRUNCATE t;
          * ROLLBACK TO save; -- rd_newRelfilenode is now forgotten
          */
         SubTransactionId rd_createSubid;    /* rel was created in current xact */
         SubTransactionId rd_newRelfilenodeSubid;    /* new relfilenode assigned in
                                                      * current xact */
     
         Form_pg_class rd_rel;       /* RELATION tuple */
         TupleDesc   rd_att;         /* tuple descriptor */
         Oid         rd_id;          /* relation's object id */
         LockInfoData rd_lockInfo;   /* lock mgr's info for locking relation */
         RuleLock   *rd_rules;       /* rewrite rules */
         MemoryContext rd_rulescxt;  /* private memory cxt for rd_rules, if any */
         TriggerDesc *trigdesc;      /* Trigger info, or NULL if rel has none */
         /* use "struct" here to avoid needing to include rowsecurity.h: */
         struct RowSecurityDesc *rd_rsdesc;  /* row security policies, or NULL */
     
         /* data managed by RelationGetFKeyList: */
         List       *rd_fkeylist;    /* list of ForeignKeyCacheInfo (see below) */
         bool        rd_fkeyvalid;   /* true if list has been computed */
     
         MemoryContext rd_partkeycxt;    /* private memory cxt for the below */
         struct PartitionKeyData *rd_partkey;    /* partition key, or NULL */
         MemoryContext rd_pdcxt;     /* private context for partdesc */
         struct PartitionDescData *rd_partdesc;  /* partitions, or NULL */
         List       *rd_partcheck;   /* partition CHECK quals */
     
         /* data managed by RelationGetIndexList: */
         List       *rd_indexlist;   /* list of OIDs of indexes on relation */
         Oid         rd_oidindex;    /* OID of unique index on OID, if any */
         Oid         rd_pkindex;     /* OID of primary key, if any */
         Oid         rd_replidindex; /* OID of replica identity index, if any */
     
         /* data managed by RelationGetStatExtList: */
         List       *rd_statlist;    /* list of OIDs of extended stats */
     
         /* data managed by RelationGetIndexAttrBitmap: */
         Bitmapset  *rd_indexattr;   /* columns used in non-projection indexes */
         Bitmapset  *rd_projindexattr;   /* columns used in projection indexes */
         Bitmapset  *rd_keyattr;     /* cols that can be ref'd by foreign keys */
         Bitmapset  *rd_pkattr;      /* cols included in primary key */
         Bitmapset  *rd_idattr;      /* included in replica identity index */
         Bitmapset  *rd_projidx;     /* Oids of projection indexes */
     
         PublicationActions *rd_pubactions;  /* publication actions */
     
         /*
          * rd_options is set whenever rd_rel is loaded into the relcache entry.
          * Note that you can NOT look into rd_rel for this data.  NULL means "use
          * defaults".
          */
         bytea      *rd_options;     /* parsed pg_class.reloptions */
     
         /* These are non-NULL only for an index relation: */
         Form_pg_index rd_index;     /* pg_index tuple describing this index */
         /* use "struct" here to avoid needing to include htup.h: */
         struct HeapTupleData *rd_indextuple;    /* all of pg_index tuple */
     
         /*
          * index access support info (used only for an index relation)
          *
          * Note: only default support procs for each opclass are cached, namely
          * those with lefttype and righttype equal to the opclass's opcintype. The
          * arrays are indexed by support function number, which is a sufficient
          * identifier given that restriction.
          *
          * Note: rd_amcache is available for index AMs to cache private data about
          * an index.  This must be just a cache since it may get reset at any time
          * (in particular, it will get reset by a relcache inval message for the
          * index).  If used, it must point to a single memory chunk palloc'd in
          * rd_indexcxt.  A relcache reset will include freeing that chunk and
          * setting rd_amcache = NULL.
          */
         Oid         rd_amhandler;   /* OID of index AM's handler function */
         MemoryContext rd_indexcxt;  /* private memory cxt for this stuff */
         /* use "struct" here to avoid needing to include amapi.h: */
         struct IndexAmRoutine *rd_amroutine;    /* index AM's API struct */
         Oid        *rd_opfamily;    /* OIDs of op families for each index col */
         Oid        *rd_opcintype;   /* OIDs of opclass declared input data types */
         RegProcedure *rd_support;   /* OIDs of support procedures */
         FmgrInfo   *rd_supportinfo; /* lookup info for support procedures */
         int16      *rd_indoption;   /* per-column AM-specific flags */
         List       *rd_indexprs;    /* index expression trees, if any */
         List       *rd_indpred;     /* index predicate tree, if any */
         Oid        *rd_exclops;     /* OIDs of exclusion operators, if any */
         Oid        *rd_exclprocs;   /* OIDs of exclusion ops' procs, if any */
         uint16     *rd_exclstrats;  /* exclusion ops' strategy numbers, if any */
         void       *rd_amcache;     /* available for use by index AM */
         Oid        *rd_indcollation;    /* OIDs of index collations */
     
         /*
          * foreign-table support
          *
          * rd_fdwroutine must point to a single memory chunk palloc'd in
          * CacheMemoryContext.  It will be freed and reset to NULL on a relcache
          * reset.
          */
     
         /* use "struct" here to avoid needing to include fdwapi.h: */
         struct FdwRoutine *rd_fdwroutine;   /* cached function pointers, or NULL */
     
         /*
          * Hack for CLUSTER, rewriting ALTER TABLE, etc: when writing a new
          * version of a table, we need to make any toast pointers inserted into it
          * have the existing toast table's OID, not the OID of the transient toast
          * table.  If rd_toastoid isn't InvalidOid, it is the OID to place in
          * toast pointers inserted into this rel.  (Note it's set on the new
          * version of the main heap, not the toast table itself.)  This also
          * causes toast_save_datum() to try to preserve toast value OIDs.
          */
         Oid         rd_toastoid;    /* Real TOAST table's OID, or InvalidOid */
     
         /* use "struct" here to avoid needing to include pgstat.h: */
         struct PgStat_TableStatus *pgstat_info; /* statistics collection area */
     } RelationData;
      
    typedef struct RelationData *Relation;
     
    

**IndexOptInfo**  
索引信息

    
    
     /*
      * IndexOptInfo
      *      Per-index information for planning/optimization
      *
      *      indexkeys[], indexcollations[] each have ncolumns entries.
      *      opfamily[], and opcintype[] each have nkeycolumns entries. They do
      *      not contain any information about included attributes.
      *
      *      sortopfamily[], reverse_sort[], and nulls_first[] have
      *      nkeycolumns entries, if the index is ordered; but if it is unordered,
      *      those pointers are NULL.
      *
      *      Zeroes in the indexkeys[] array indicate index columns that are
      *      expressions; there is one element in indexprs for each such column.
      *
      *      For an ordered index, reverse_sort[] and nulls_first[] describe the
      *      sort ordering of a forward indexscan; we can also consider a backward
      *      indexscan, which will generate the reverse ordering.
      *
      *      The indexprs and indpred expressions have been run through
      *      prepqual.c and eval_const_expressions() for ease of matching to
      *      WHERE clauses. indpred is in implicit-AND form.
      *
      *      indextlist is a TargetEntry list representing the index columns.
      *      It provides an equivalent base-relation Var for each simple column,
      *      and links to the matching indexprs element for each expression column.
      *
      *      While most of these fields are filled when the IndexOptInfo is created
      *      (by plancat.c), indrestrictinfo and predOK are set later, in
      *      check_index_predicates().
      */
     typedef struct IndexOptInfo
     {
         NodeTag     type;
     
         Oid         indexoid;       /* Index的OID,OID of the index relation */
         Oid         reltablespace;  /* Index的表空间,tablespace of index (not table) */
         RelOptInfo *rel;            /* 指向Relation的指针,back-link to index's table */
     
         /* index-size statistics (from pg_class and elsewhere) */
         BlockNumber pages;          /* Index的pages,number of disk pages in index */
         double      tuples;         /* Index的元组数,number of index tuples in index */
         int         tree_height;    /* 索引高度,index tree height, or -1 if unknown */
     
         /* index descriptor information */
         int         ncolumns;       /* 索引的列数,number of columns in index */
         int         nkeycolumns;    /* 索引的关键列数,number of key columns in index */
         int        *indexkeys;      /* column numbers of index's attributes both
                                      * key and included columns, or 0 */
         Oid        *indexcollations;    /* OIDs of collations of index columns */
         Oid        *opfamily;       /* OIDs of operator families for columns */
         Oid        *opcintype;      /* OIDs of opclass declared input data types */
         Oid        *sortopfamily;   /* OIDs of btree opfamilies, if orderable */
         bool       *reverse_sort;   /* 倒序?is sort order descending? */
         bool       *nulls_first;    /* NULLs值优先?do NULLs come first in the sort order? */
         bool       *canreturn;      /* 索引列可通过Index-Only Scan返回?which index cols can be returned in an
                                      * index-only scan? */
         Oid         relam;          /* 访问方法OID,OID of the access method (in pg_am) */
     
         List       *indexprs;       /* 非简单索引列表达式链表,如函数索引,expressions for non-simple index columns */
         List       *indpred;        /* predicate if a partial index, else NIL */
     
         List       *indextlist;     /* 投影列?targetlist representing index columns */
     
         List       *indrestrictinfo;    /* 索引约束条件,parent relation's baserestrictinfo
                                          * list, less any conditions implied by
                                          * the index's predicate (unless it's a
                                          * target rel, see comments in
                                          * check_index_predicates()) */
     
         bool        predOK;         /* True,如索引前导满足查询要求,true if index predicate matches query */
         bool        unique;         /* 唯一索引?true if a unique index */
         bool        immediate;      /* 唯一性校验是否立即生效?is uniqueness enforced immediately? */
         bool        hypothetical;   /* 虚拟索引?true if index doesn't really exist */
     
         /* Remaining fields are copied from the index AM's API struct: */
         //从Index Relation拷贝过来的AM(访问方法)API信息
         bool        amcanorderbyop; /* does AM support order by operator result? */
         bool        amoptionalkey;  /* can query omit key for the first column? */
         bool        amsearcharray;  /* can AM handle ScalarArrayOpExpr quals? */
         bool        amsearchnulls;  /* can AM search for NULL/NOT NULL entries? */
         bool        amhasgettuple;  /* does AM have amgettuple interface? */
         bool        amhasgetbitmap; /* does AM have amgetbitmap interface? */
         bool        amcanparallel;  /* does AM support parallel scan? */
         /* Rather than include amapi.h here, we declare amcostestimate like this */
         void        (*amcostestimate) ();   /* 访问方法的估算函数,AM's cost estimator */
     } IndexOptInfo;
     
    

**ForeignKeyOptInfo**  
外键优化信息

    
    
     /*
      * ForeignKeyOptInfo
      *      Per-foreign-key information for planning/optimization
      *
      * The per-FK-column arrays can be fixed-size because we allow at most
      * INDEX_MAX_KEYS columns in a foreign key constraint.  Each array has
      * nkeys valid entries.
      */
     typedef struct ForeignKeyOptInfo
     {
         NodeTag     type;
     
         /* Basic data about the foreign key (fetched from catalogs): */
         Index       con_relid;      /* RT index of the referencing table */
         Index       ref_relid;      /* RT index of the referenced table */
         int         nkeys;          /* number of columns in the foreign key */
         AttrNumber  conkey[INDEX_MAX_KEYS]; /* cols in referencing table */
         AttrNumber  confkey[INDEX_MAX_KEYS];    /* cols in referenced table */
         Oid         conpfeqop[INDEX_MAX_KEYS];  /* PK = FK operator OIDs */
     
         /* Derived info about whether FK's equality conditions match the query: */
         int         nmatched_ec;    /* # of FK cols matched by ECs */
         int         nmatched_rcols; /* # of FK cols matched by non-EC rinfos */
         int         nmatched_ri;    /* total # of non-EC rinfos matched to FK */
         /* Pointer to eclass matching each column's condition, if there is one */
         struct EquivalenceClass *eclass[INDEX_MAX_KEYS];
         /* List of non-EC RestrictInfos matching each column's condition */
         List       *rinfos[INDEX_MAX_KEYS];
     } ForeignKeyOptInfo;
     
    

**StatisticExtInfo**

    
    
     /*
      * StatisticExtInfo
      *      Information about extended statistics for planning/optimization
      *
      * Each pg_statistic_ext row is represented by one or more nodes of this
      * type, or even zero if ANALYZE has not computed them.
      */
     typedef struct StatisticExtInfo
     {
         NodeTag     type;
     
         Oid         statOid;        /* OID of the statistics row */
         RelOptInfo *rel;            /* back-link to statistic's table */
         char        kind;           /* statistic kind of this entry */
         Bitmapset  *keys;           /* attnums of the columns covered */
     } StatisticExtInfo;
    

### 二、源码解读

add_base_rels_to_query函数构建查询的RelOptInfos  
**add_base_rels_to_query**

    
    
    /*
      * add_base_rels_to_query
      *
      *    Scan the query's jointree and create baserel RelOptInfos for all
      *    the base relations (ie, table, subquery, and function RTEs)
      *    appearing in the jointree.
      *
      * The initial invocation must pass root->parse->jointree as the value of
      * jtnode.  Internally, the function recurses through the jointree.
      *
      * At the end of this process, there should be one baserel RelOptInfo for
      * every non-join RTE that is used in the query.  Therefore, this routine
      * is the only place that should call build_simple_rel with reloptkind
      * RELOPT_BASEREL.  (Note: build_simple_rel recurses internally to build
      * "other rel" RelOptInfos for the members of any appendrels we find here.)
      */
     void
     add_base_rels_to_query(PlannerInfo *root, Node *jtnode)//遍历jointree递归实现
     {
       //以下的递归遍历结构先前的内容已反复出现N次
         if (jtnode == NULL)
             return;
         if (IsA(jtnode, RangeTblRef))//RTR
         {
             int         varno = ((RangeTblRef *) jtnode)->rtindex;
     
             (void) build_simple_rel(root, varno, NULL);
         }
         else if (IsA(jtnode, FromExpr))//FromExpr
         {
             FromExpr   *f = (FromExpr *) jtnode;
             ListCell   *l;
     
             foreach(l, f->fromlist)//fromlist
                 add_base_rels_to_query(root, lfirst(l));
         }
         else if (IsA(jtnode, JoinExpr))//JoinExpr
         {
             JoinExpr   *j = (JoinExpr *) jtnode;
     
             add_base_rels_to_query(root, j->larg);//左Child
             add_base_rels_to_query(root, j->rarg);//右Child
         }
         else
             elog(ERROR, "unrecognized node type: %d",
                  (int) nodeTag(jtnode));
     }
     
     /*
      * build_simple_rel
      *    Construct a new RelOptInfo for a base relation or 'other' relation.
      */
     RelOptInfo *
     build_simple_rel(PlannerInfo *root, int relid, RelOptInfo *parent)
     {
         RelOptInfo *rel;
         RangeTblEntry *rte;
     
         /* Rel should not exist already */
         Assert(relid > 0 && relid < root->simple_rel_array_size);
         if (root->simple_rel_array[relid] != NULL)
             elog(ERROR, "rel %d already exists", relid);
     
         /* Fetch RTE for relation */
         rte = root->simple_rte_array[relid];//获取RTE
         Assert(rte != NULL);
     
         rel = makeNode(RelOptInfo);//构建RelOptInfo
         rel->reloptkind = parent ? RELOPT_OTHER_MEMBER_REL : RELOPT_BASEREL;
         rel->relids = bms_make_singleton(relid);//初始化relids
         rel->rows = 0;
         /* cheap startup cost is interesting iff not all tuples to be retrieved */
         rel->consider_startup = (root->tuple_fraction > 0);
         rel->consider_param_startup = false;    /* might get changed later */
         rel->consider_parallel = false; /* might get changed later */
         rel->reltarget = create_empty_pathtarget();
         rel->pathlist = NIL;
         rel->ppilist = NIL;
         rel->partial_pathlist = NIL;
         rel->cheapest_startup_path = NULL;
         rel->cheapest_total_path = NULL;
         rel->cheapest_unique_path = NULL;
         rel->cheapest_parameterized_paths = NIL;
         rel->direct_lateral_relids = NULL;
         rel->lateral_relids = NULL;
         rel->relid = relid;
         rel->rtekind = rte->rtekind;
         /* min_attr, max_attr, attr_needed, attr_widths are set below */
         rel->lateral_vars = NIL;
         rel->lateral_referencers = NULL;
         rel->indexlist = NIL;
         rel->statlist = NIL;
         rel->pages = 0;
         rel->tuples = 0;
         rel->allvisfrac = 0;
         rel->subroot = NULL;
         rel->subplan_params = NIL;
         rel->rel_parallel_workers = -1; /* set up in get_relation_info */
         rel->serverid = InvalidOid;
         rel->userid = rte->checkAsUser;
         rel->useridiscurrent = false;
         rel->fdwroutine = NULL;
         rel->fdw_private = NULL;
         rel->unique_for_rels = NIL;
         rel->non_unique_for_rels = NIL;
         rel->baserestrictinfo = NIL;
         rel->baserestrictcost.startup = 0;
         rel->baserestrictcost.per_tuple = 0;
         rel->baserestrict_min_security = UINT_MAX;
         rel->joininfo = NIL;
         rel->has_eclass_joins = false;
         rel->consider_partitionwise_join = false; /* might get changed later */
         rel->part_scheme = NULL;
         rel->nparts = 0;
         rel->boundinfo = NULL;
         rel->partition_qual = NIL;
         rel->part_rels = NULL;
         rel->partexprs = NULL;
         rel->nullable_partexprs = NULL;
         rel->partitioned_child_rels = NIL;
     
         /*
          * Pass top parent's relids down the inheritance hierarchy. If the parent
          * has top_parent_relids set, it's a direct or an indirect child of the
          * top parent indicated by top_parent_relids. By extension this child is
          * also an indirect child of that parent.
          */
         if (parent)//存在父RelOptInfo,设置top_parent_relids变量(最上层的Relids)
         {
             if (parent->top_parent_relids)
                 rel->top_parent_relids = parent->top_parent_relids;
             else
                 rel->top_parent_relids = bms_copy(parent->relids);
         }
         else
             rel->top_parent_relids = NULL;
     
         /* Check type of rtable entry */
         switch (rte->rtekind)
         {
             case RTE_RELATION:
                 /* Table --- retrieve statistics from the system catalogs */
                 get_relation_info(root, rte->relid, rte->inh, rel);//基表,从数据字典中获取统计信息
                 break;
             case RTE_SUBQUERY:
             case RTE_FUNCTION:
             case RTE_TABLEFUNC:
             case RTE_VALUES:
             case RTE_CTE:
             case RTE_NAMEDTUPLESTORE:
     
                 /*
            * 子查询/函数/tablefunc/values lis/CTE/ENR,设置属性范围&数组
                  * Subquery, function, tablefunc, values list, CTE, or ENR --- set
                  * up attr range and arrays
                  *
                  * Note: 0 is included in range to support whole-row Vars
                  */
                 rel->min_attr = 0;
                 rel->max_attr = list_length(rte->eref->colnames);
                 rel->attr_needed = (Relids *)
                     palloc0((rel->max_attr - rel->min_attr + 1) * sizeof(Relids));
                 rel->attr_widths = (int32 *)
                     palloc0((rel->max_attr - rel->min_attr + 1) * sizeof(int32));
                 break;
             default:
                 elog(ERROR, "unrecognized RTE kind: %d",
                      (int) rte->rtekind);
                 break;
         }
     
         /* Save the finished struct in the query's simple_rel_array */
         root->simple_rel_array[relid] = rel;//存储RelOptInfo
     
         /*
          * This is a convenient spot at which to note whether rels participating
          * in the query have any securityQuals attached.  If so, increase
          * root->qual_security_level to ensure it's larger than the maximum
          * security level needed for securityQuals.
          */
         if (rte->securityQuals)
             root->qual_security_level = Max(root->qual_security_level,
                                             list_length(rte->securityQuals));
     
         /*
          * If this rel is an appendrel parent, recurse to build "other rel"
          * RelOptInfos for its children.  They are "other rels" because they are
          * not in the main join tree, but we will need RelOptInfos to plan access
          * to them.
          */
       //如果这个RelOptInfo是一个appendrel的父节点,递归的构建其children对应的RelOptInfos
         if (rte->inh)
         {
             ListCell   *l;
             int         nparts = rel->nparts;
             int         cnt_parts = 0;
     
             if (nparts > 0)
                 rel->part_rels = (RelOptInfo **)
                     palloc(sizeof(RelOptInfo *) * nparts);
     
             foreach(l, root->append_rel_list)//递归调用
             {
                 AppendRelInfo *appinfo = (AppendRelInfo *) lfirst(l);
                 RelOptInfo *childrel;
     
                 /* append_rel_list contains all append rels; ignore others */
                 if (appinfo->parent_relid != relid)
                     continue;
     
                 childrel = build_simple_rel(root, appinfo->child_relid,
                                             rel);
     
                 /* Nothing more to do for an unpartitioned table. */
                 if (!rel->part_scheme)
                     continue;
     
                 /*
                  * The order of partition OIDs in append_rel_list is the same as
                  * the order in the PartitionDesc, so the order of part_rels will
                  * also match the PartitionDesc.  See expand_partitioned_rtentry.
                  */
                 Assert(cnt_parts < nparts);
                 rel->part_rels[cnt_parts] = childrel;
                 cnt_parts++;
             }
     
             /* We should have seen all the child partitions. */
             Assert(cnt_parts == nparts);
         }
     
         return rel;
     }
    
    /*
      * get_relation_info -      *    Retrieves catalog information for a given relation.
      *
      * Given the Oid of the relation, return the following info into fields
      * of the RelOptInfo struct:
      *
      *  min_attr    lowest valid AttrNumber  最小有效属性编号
      *  max_attr    highest valid AttrNumber 最大有效属性编号
      *  indexlist   list of IndexOptInfos for relation's indexes 索引的IndexOptInfo链表 
      *  statlist    list of StatisticExtInfo for relation's statistic objects  扩展统计信息链表
      *  serverid    if it's a foreign table, the server OID  FDW所在服务器ID
      *  fdwroutine  if it's a foreign table, the FDW function pointers FDW函数指针
      *  pages       number of pages  pages数
      *  tuples      number of tuples 元组数
      *  rel_parallel_workers user-defined number of parallel workers 用户自定义的并行worker数
      *
      * Also, add information about the relation's foreign keys to root->fkey_list.
      * 
      * Relation的外键信息会添加到root->fkey_list中
      *
      * Also, initialize the attr_needed[] and attr_widths[] arrays.  In most
      * cases these are left as zeroes, but sometimes we need to compute attr
      * widths here, and we may as well cache the results for costsize.c.
      *
      * If inhparent is true, all we need to do is set up the attr arrays:
      * the RelOptInfo actually represents the appendrel formed by an inheritance
      * tree, and so the parent rel's physical size and index information isn't
      * important for it.
      */
     void
     get_relation_info(PlannerInfo *root, Oid relationObjectId, bool inhparent,
                       RelOptInfo *rel)
     {
         Index       varno = rel->relid;//Relation的relid
         Relation    relation;//Relation信息
         bool        hasindex;//是否含有index
         List       *indexinfos = NIL;//IndexOptInfo链表
     
         /*
          * We need not lock the relation since it was already locked, either by
          * the rewriter or when expand_inherited_rtentry() added it to the query's
          * rangetable.
          */
         relation = heap_open(relationObjectId, NoLock);//Relation信息
     
         /* Temporary and unlogged relations are inaccessible during recovery. */
         if (!RelationNeedsWAL(relation) && RecoveryInProgress())//恢复过程不允许访问
             ereport(ERROR,
                     (errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
                      errmsg("cannot access temporary or unlogged relations during recovery")));
     
         rel->min_attr = FirstLowInvalidHeapAttributeNumber + 1;//(FirstLowInvalidHeapAttributeNumber=-8)
       //#define RelationGetNumberOfAttributes(relation) ((relation)->rd_rel->relnatts)
         rel->max_attr = RelationGetNumberOfAttributes(relation);//
         rel->reltablespace = RelationGetForm(relation)->reltablespace;//表空间
     
         Assert(rel->max_attr >= rel->min_attr);
         rel->attr_needed = (Relids *)
             palloc0((rel->max_attr - rel->min_attr + 1) * sizeof(Relids));//初始化
         rel->attr_widths = (int32 *)
             palloc0((rel->max_attr - rel->min_attr + 1) * sizeof(int32));//初始化
     
         /*
          * Estimate relation size --- unless it's an inheritance parent, in which
          * case the size will be computed later in set_append_rel_pathlist, and we
          * must leave it zero for now to avoid bollixing the total_table_pages
          * calculation.
          */
       //如果不是inheritance parent,则估算Relation的大小
         if (!inhparent)
             estimate_rel_size(relation, rel->attr_widths - rel->min_attr,
                               &rel->pages, &rel->tuples, &rel->allvisfrac);
     
         /* Retrieve the parallel_workers reloption, or -1 if not set. */
         /*
       #define RelationGetParallelWorkers(relation, defaultpw) \
         ((relation)->rd_options ? \
          ((StdRdOptions *) (relation)->rd_options)->parallel_workers : (defaultpw))
       */
         rel->rel_parallel_workers = RelationGetParallelWorkers(relation, -1);
     
         /*
          * Make list of indexes.  Ignore indexes on system catalogs if told to.
          * Don't bother with indexes for an inheritance parent, either.
          */
         if (inhparent ||
             (IgnoreSystemIndexes && IsSystemRelation(relation)))//继承表/系统表并且忽略索引
             hasindex = false;
         else
             hasindex = relation->rd_rel->relhasindex;//是否含有索引
     
         if (hasindex)//存在索引,则生成IndexOptInfo链表
         {
             List       *indexoidlist;
             ListCell   *l;
             LOCKMODE    lmode;
     
             indexoidlist = RelationGetIndexList(relation);//获取Relation的Index Oid链表
     
             /*
              * For each index, we get the same type of lock that the executor will
              * need, and do not release it.  This saves a couple of trips to the
              * shared lock manager while not creating any real loss of
              * concurrency, because no schema changes could be happening on the
              * index while we hold lock on the parent rel, and neither lock type
              * blocks any other kind of index operation.
              */
             if (rel->relid == root->parse->resultRelation)
                 lmode = RowExclusiveLock;//该Relation是结果Relation,锁模式为行排它锁
             else
                 lmode = AccessShareLock;//否则为访问共享锁
     
             foreach(l, indexoidlist)//遍历Index Oid
             {
                 Oid         indexoid = lfirst_oid(l);
                 Relation    indexRelation;
                 Form_pg_index index;
                 IndexAmRoutine *amroutine;
                 IndexOptInfo *info;
                 int         ncolumns,
                             nkeycolumns;
                 int         i;
     
                 /*
                  * Extract info from the relation descriptor for the index.
                  */
                 indexRelation = index_open(indexoid, lmode);//获取Index相关信息
                 index = indexRelation->rd_index;
     
                 /*
                  * Ignore invalid indexes, since they can't safely be used for
                  * queries.  Note that this is OK because the data structure we
                  * are constructing is only used by the planner --- the executor
                  * still needs to insert into "invalid" indexes, if they're marked
                  * IndexIsReady.
                  */
                 if (!IndexIsValid(index))
                 {
                     index_close(indexRelation, NoLock);//忽略无效的Index
                     continue;
                 }
     
                 /*
                  * Ignore partitioned indexes, since they are not usable for
                  * queries.
                  */
                 if (indexRelation->rd_rel->relkind == RELKIND_PARTITIONED_INDEX)
                 {
                     index_close(indexRelation, NoLock);//忽略分区索引
                     continue;
                 }
     
                 /*
                  * If the index is valid, but cannot yet be used, ignore it; but
                  * mark the plan we are generating as transient. See
                  * src/backend/access/heap/README.HOT for discussion.
                  */
                 if (index->indcheckxmin &&
                     !TransactionIdPrecedes(HeapTupleHeaderGetXmin(indexRelation->rd_indextuple->t_data),
                                            TransactionXmin))
                 {
                     root->glob->transientPlan = true;//有效索引,但还不能正常使用,忽略之
                     index_close(indexRelation, NoLock);
                     continue;
                 }
     
                 info = makeNode(IndexOptInfo);//创建IndexOptInfo节点
     
                 info->indexoid = index->indexrelid;//OID
                 info->reltablespace =
                     RelationGetForm(indexRelation)->reltablespace;//表空间
                 info->rel = rel;//Index所在的Relation
                 info->ncolumns = ncolumns = index->indnatts;//Index的列个数
                 info->nkeycolumns = nkeycolumns = index->indnkeyatts;//
     
                 info->indexkeys = (int *) palloc(sizeof(int) * ncolumns);//初始化内存空间
                 info->indexcollations = (Oid *) palloc(sizeof(Oid) * nkeycolumns);
                 info->opfamily = (Oid *) palloc(sizeof(Oid) * nkeycolumns);
                 info->opcintype = (Oid *) palloc(sizeof(Oid) * nkeycolumns);
                 info->canreturn = (bool *) palloc(sizeof(bool) * ncolumns);
     
                 for (i = 0; i < ncolumns; i++)//索引键
                 {
                     info->indexkeys[i] = index->indkey.values[i];
                     info->canreturn[i] = index_can_return(indexRelation, i + 1);
                 }
     
                 for (i = 0; i < nkeycolumns; i++)//索引键属性
                 {
                     info->opfamily[i] = indexRelation->rd_opfamily[i];
                     info->opcintype[i] = indexRelation->rd_opcintype[i];
                     info->indexcollations[i] = indexRelation->rd_indcollation[i];
                 }
     
                 info->relam = indexRelation->rd_rel->relam;//?
     
                 /* We copy just the fields we need, not all of rd_amroutine */
                 amroutine = indexRelation->rd_amroutine;//拷贝IndexRelation中的信息
                 info->amcanorderbyop = amroutine->amcanorderbyop;
                 info->amoptionalkey = amroutine->amoptionalkey;
                 info->amsearcharray = amroutine->amsearcharray;
                 info->amsearchnulls = amroutine->amsearchnulls;
                 info->amcanparallel = amroutine->amcanparallel;
                 info->amhasgettuple = (amroutine->amgettuple != NULL);
                 info->amhasgetbitmap = (amroutine->amgetbitmap != NULL);
                 info->amcostestimate = amroutine->amcostestimate;
                 Assert(info->amcostestimate != NULL);
     
                 /*
                  * Fetch the ordering information for the index, if any.
                  */
                 if (info->relam == BTREE_AM_OID)//BTree
                 {
                     /*
                      * If it's a btree index, we can use its opfamily OIDs
                      * directly as the sort ordering opfamily OIDs.
                      */
                     Assert(amroutine->amcanorder);
     
                     info->sortopfamily = info->opfamily;
                     info->reverse_sort = (bool *) palloc(sizeof(bool) * nkeycolumns);
                     info->nulls_first = (bool *) palloc(sizeof(bool) * nkeycolumns);
     
                     for (i = 0; i < nkeycolumns; i++)
                     {
                         int16       opt = indexRelation->rd_indoption[i];
     
                         info->reverse_sort[i] = (opt & INDOPTION_DESC) != 0;
                         info->nulls_first[i] = (opt & INDOPTION_NULLS_FIRST) != 0;
                     }
                 }
                 else if (amroutine->amcanorder)//可排序的访问方法
                 {
                     /*
                      * Otherwise, identify the corresponding btree opfamilies by
                      * trying to map this index's "<" operators into btree.  Since
                      * "<" uniquely defines the behavior of a sort order, this is
                      * a sufficient test.
                      *
                      * XXX This method is rather slow and also requires the
                      * undesirable assumption that the other index AM numbers its
                      * strategies the same as btree.  It'd be better to have a way
                      * to explicitly declare the corresponding btree opfamily for
                      * each opfamily of the other index type.  But given the lack
                      * of current or foreseeable amcanorder index types, it's not
                      * worth expending more effort on now.
                      */
                     info->sortopfamily = (Oid *) palloc(sizeof(Oid) * nkeycolumns);
                     info->reverse_sort = (bool *) palloc(sizeof(bool) * nkeycolumns);
                     info->nulls_first = (bool *) palloc(sizeof(bool) * nkeycolumns);
     
                     for (i = 0; i < nkeycolumns; i++)
                     {
                         int16       opt = indexRelation->rd_indoption[i];
                         Oid         ltopr;
                         Oid         btopfamily;
                         Oid         btopcintype;
                         int16       btstrategy;
     
                         info->reverse_sort[i] = (opt & INDOPTION_DESC) != 0;//是否倒序?
                         info->nulls_first[i] = (opt & INDOPTION_NULLS_FIRST) != 0;//NULL值优先?
     
                         ltopr = get_opfamily_member(info->opfamily[i],
                                                     info->opcintype[i],
                                                     info->opcintype[i],
                                                     BTLessStrategyNumber);
                         if (OidIsValid(ltopr) &&
                             get_ordering_op_properties(ltopr,
                                                        &btopfamily,
                                                        &btopcintype,
                                                        &btstrategy) &&
                             btopcintype == info->opcintype[i] &&
                             btstrategy == BTLessStrategyNumber)
                         {
                             /* Successful mapping */
                             info->sortopfamily[i] = btopfamily;//排序操作类?
                         }
                         else//失败,索引视为未排序
                         {
                             /* Fail ... quietly treat index as unordered */
                             info->sortopfamily = NULL;
                             info->reverse_sort = NULL;
                             info->nulls_first = NULL;
                             break;
                         }
                     }
                 }
                 else//非可排序,设置为NULL
                 {
                     info->sortopfamily = NULL;
                     info->reverse_sort = NULL;
                     info->nulls_first = NULL;
                 }
     
                 /*
                  * Fetch the index expressions and predicate, if any.  We must
                  * modify the copies we obtain from the relcache to have the
                  * correct varno for the parent relation, so that they match up
                  * correctly against qual clauses.
                  */
                 info->indexprs = RelationGetIndexExpressions(indexRelation);//索引表达式(函数索引)
                 info->indpred = RelationGetIndexPredicate(indexRelation);//索引谓词信息(条件索引)
                 if (info->indexprs && varno != 1)
                     ChangeVarNodes((Node *) info->indexprs, 1, varno, 0);
                 if (info->indpred && varno != 1)
                     ChangeVarNodes((Node *) info->indpred, 1, varno, 0);
     
                 /* Build targetlist using the completed indexprs data */
                 info->indextlist = build_index_tlist(root, info, relation);//索引的列
     
                 info->indrestrictinfo = NIL;    /* set later, in indxpath.c */
                 info->predOK = false;   /* set later, in indxpath.c */
                 info->unique = index->indisunique;
                 info->immediate = index->indimmediate;
                 info->hypothetical = false;
     
                 /*
                  * Estimate the index size.  If it's not a partial index, we lock
                  * the number-of-tuples estimate to equal the parent table; if it
                  * is partial then we have to use the same methods as we would for
                  * a table, except we can be sure that the index is not larger
                  * than the table.
                  */
                 if (info->indpred == NIL)//非条件索引
                 {
                     info->pages = RelationGetNumberOfBlocks(indexRelation);//Index的pages
                     info->tuples = rel->tuples;//Index的元组
                 }
                 else
                 {
                     double      allvisfrac; /* dummy */
     
                     estimate_rel_size(indexRelation, NULL,
                                       &info->pages, &info->tuples, &allvisfrac);//估算Index的大小
                     if (info->tuples > rel->tuples)//Index的元组数不能大于数据表元组数
                         info->tuples = rel->tuples;
                 }
     
                 if (info->relam == BTREE_AM_OID)//BTree
                 {
                     /* For btrees, get tree height while we have the index open */
                     info->tree_height = _bt_getrootheight(indexRelation);//BTree高度
                 }
                 else
                 {
                     /* For other index types, just set it to "unknown" for now */
                     info->tree_height = -1;//非BTree
                 }
     
                 index_close(indexRelation, NoLock);
     
                 indexinfos = lcons(info, indexinfos);
             }
     
             list_free(indexoidlist);
         }
     
         rel->indexlist = indexinfos;
     
         rel->statlist = get_relation_statistics(rel, relation);
     
         /* Grab foreign-table info using the relcache, while we have it */
         if (relation->rd_rel->relkind == RELKIND_FOREIGN_TABLE)//FDW
         {
             rel->serverid = GetForeignServerIdByRelId(RelationGetRelid(relation));
             rel->fdwroutine = GetFdwRoutineForRelation(relation, true);
         }
         else
         {
             rel->serverid = InvalidOid;
             rel->fdwroutine = NULL;
         }
     
         /* Collect info about relation's foreign keys, if relevant */
         get_relation_foreign_keys(root, rel, relation, inhparent);//收集外键信息
     
         /*
          * Collect info about relation's partitioning scheme, if any. Only
          * inheritance parents may be partitioned.
          */
         if (inhparent && relation->rd_rel->relkind == RELKIND_PARTITIONED_TABLE)
             set_relation_partition_info(root, rel, relation);//收集分区表信息
     
         heap_close(relation, NoLock);
     
         /*
          * Allow a plugin to editorialize on the info we obtained from the
          * catalogs.  Actions might include altering the assumed relation size,
          * removing an index, or adding a hypothetical index to the indexlist.
          */
         if (get_relation_info_hook)
             (*get_relation_info_hook) (root, relationObjectId, inhparent, rel);//钩子函数
     }
     
    

### 三、跟踪分析

测试脚本,创建部分(条件)索引和函数索引:

    
    
    testdb=# create index idx_dwxx_expr on t_dwxx(trim(dwmc));
    CREATE INDEX
    testdb=# create index idx_dwxx_predicate on t_dwxx(dwdz) where dwdz like '广东省%';
    CREATE INDEX
    
    testdb=# explain verbose select * from t_dwxx where dwdz like '广东省%';
                              QUERY PLAN                           
    ---------------------------------------------------------------     Seq Scan on public.t_dwxx  (cost=0.00..1.04 rows=1 width=474)
       Output: dwmc, dwbh, dwdz
       Filter: ((t_dwxx.dwdz)::text ~~ '广东省%'::text)
    (3 rows)
    
    

跟踪分析:

    
    
    (gdb) b add_base_rels_to_query
    Breakpoint 1 at 0x765400: file initsplan.c, line 107.
    (gdb) c
    Continuing.
    
    Breakpoint 1, add_base_rels_to_query (root=0x23107f8, jtnode=0x2251cc8) at initsplan.c:107
    107   if (jtnode == NULL)
    (gdb) n
    109   if (IsA(jtnode, RangeTblRef))
    (gdb) 
    115   else if (IsA(jtnode, FromExpr))
    (gdb) 
    

第一次调用,jtnode类型为FromExpr

    
    
    117     FromExpr   *f = (FromExpr *) jtnode;
    (gdb) 
    120     foreach(l, f->fromlist)
    (gdb) 
    121       add_base_rels_to_query(root, lfirst(l));
    (gdb) 
    
    Breakpoint 1, add_base_rels_to_query (root=0x23107f8, jtnode=0x22515f0) at initsplan.c:107
    107   if (jtnode == NULL)
    (gdb) 
    

第二次调用,类型为RTR

    
    
    109   if (IsA(jtnode, RangeTblRef))
    (gdb) 
    111     int     varno = ((RangeTblRef *) jtnode)->rtindex;
    (gdb) 
    113     (void) build_simple_rel(root, varno, NULL);
    (gdb) p varno
    $1 = 1
    

进入build_simple_rel

    
    
    ...
    180   switch (rte->rtekind)
    (gdb) 
    184       get_relation_info(root, rte->relid, rte->inh, rel);
    

进入get_relation_info  
查看Relation的相关信息:

    
    
    ...
    ##Relation的相关信息
    121   relation = heap_open(relationObjectId, NoLock);
    (gdb) 
    124   if (!RelationNeedsWAL(relation) && RecoveryInProgress())
    (gdb) p *relation
    $4 = {rd_node = {spcNode = 1663, dbNode = 16384, relNode = 16394}, rd_smgr = 0x230d358, rd_refcnt = 1, rd_backend = -1, 
      rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, rd_indexvalid = 1 '\001', rd_statvalid = true, 
      rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7f6b9a010380, rd_att = 0x7f6b99fff5e8, rd_id = 16394, 
      rd_lockInfo = {lockRelId = {relId = 16394, dbId = 16384}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, 
      rd_rsdesc = 0x0, rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, 
      rd_partdesc = 0x0, rd_partcheck = 0x0, rd_indexlist = 0x7f6b9a011b80, rd_oidindex = 0, rd_pkindex = 16476, 
      rd_replidindex = 16476, rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, rd_keyattr = 0x0, rd_pkattr = 0x0, 
      rd_idattr = 0x0, rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, rd_indextuple = 0x0, 
      rd_amhandler = 0, rd_indexcxt = 0x0, rd_amroutine = 0x0, rd_opfamily = 0x0, rd_opcintype = 0x0, rd_support = 0x0, 
      rd_supportinfo = 0x0, rd_indoption = 0x0, rd_indexprs = 0x0, rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, 
      rd_exclstrats = 0x0, rd_amcache = 0x0, rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, 
      pgstat_info = 0x22d11b8}
    ##rd_pkindex = 16476,关键字对应的OID
    ##pg_class输出的结构体
    (gdb) p *relation->rd_rel
    $5 = {relname = {data = "t_dwxx", '\000' <repeats 57 times>}, relnamespace = 2200, reltype = 16396, reloftype = 0, 
      relowner = 10, relam = 0, relfilenode = 16394, reltablespace = 0, relpages = 1, reltuples = 3, relallvisible = 0, 
      reltoastrelid = 0, relhasindex = true, relisshared = false, relpersistence = 112 'p', relkind = 114 'r', relnatts = 3, 
      relchecks = 0, relhasoids = false, relhasrules = false, relhastriggers = false, relhassubclass = false, 
      relrowsecurity = false, relforcerowsecurity = false, relispopulated = true, relreplident = 100 'd', 
      relispartition = false, relrewrite = 0, relfrozenxid = 587, relminmxid = 1}
    ##属性(3个)
    (gdb) p *relation->rd_att
    $6 = {natts = 3, tdtypeid = 16396, tdtypmod = -1, tdhasoid = false, tdrefcount = 1, constr = 0x7f6b99fffb18, 
      attrs = 0x7f6b99fff608}
    (gdb) p relation->rd_att->attrs[0]
    $8 = {attrelid = 16394, attname = {data = "dwmc", '\000' <repeats 59 times>}, atttypid = 1043, attstattarget = -1, 
      attlen = -1, attnum = 1, attndims = 0, attcacheoff = 0, atttypmod = 104, attbyval = false, attstorage = 120 'x', 
      attalign = 105 'i', attnotnull = false, atthasdef = false, atthasmissing = false, attidentity = 0 '\000', 
      attisdropped = false, attislocal = true, attinhcount = 0, attcollation = 100}
    (gdb) p relation->rd_att->attrs[1]
    $9 = {attrelid = 16394, attname = {data = "dwbh", '\000' <repeats 59 times>}, atttypid = 1043, attstattarget = -1, 
      attlen = -1, attnum = 2, attndims = 0, attcacheoff = -1, atttypmod = 14, attbyval = false, attstorage = 120 'x', 
      attalign = 105 'i', attnotnull = true, atthasdef = false, atthasmissing = false, attidentity = 0 '\000', 
      attisdropped = false, attislocal = true, attinhcount = 0, attcollation = 100}
    (gdb) p relation->rd_att->attrs[3]
    $10 = {attrelid = 0, attname = {data = '\000' <repeats 44 times>, "\230\023-\002", '\000' <repeats 15 times>}, 
      atttypid = 0, attstattarget = 0, attlen = 0, attnum = 0, attndims = 0, attcacheoff = 0, atttypmod = 0, attbyval = false, 
      attstorage = 0 '\000', attalign = 0 '\000', attnotnull = false, atthasdef = false, atthasmissing = false, 
      attidentity = 0 '\000', attisdropped = false, attislocal = false, attinhcount = 0, attcollation = 0}
    ##Index相应的OID
    (gdb) p relation->rd_indexlist->head->data.oid_value
    $12 = 16476
    (gdb) p relation->rd_indexlist->head->next->data.oid_value
    $13 = 16497
    (gdb) p relation->rd_indexlist->head->next->next->data.oid_value
    $14 = 16499
    ...
    

进入estimate_rel_size

    
    
    146     estimate_rel_size(relation, rel->attr_widths - rel->min_attr,
    (gdb) step
    estimate_rel_size (rel=0x7f6b9a00f390, attr_widths=0x231c674, pages=0x23124d8, tuples=0x23124e0, allvisfrac=0x23124e8)
        at plancat.c:948
    948   switch (rel->rd_rel->relkind)
    (gdb) p reltuples
    $19 = 3
    ...
    

回到get_relation_info

    
    
    (gdb) 
    get_relation_info (root=0x23107f8, relationObjectId=16394, inhparent=false, rel=0x2312428) at plancat.c:150
    150   rel->rel_parallel_workers = RelationGetParallelWorkers(relation, -1);
    
    

获取索引信息(IndexOptInfo的获取在这里是重点)

    
    
    162   if (hasindex)
    (gdb) 
    168     indexoidlist = RelationGetIndexList(relation);
    ...
    #第一个Index
    (gdb) p indexoid
    $2 = 16476
    #IndexRelation的相关信息
    (gdb) p *indexRelation
    $3 = {rd_node = {spcNode = 1663, dbNode = 16384, relNode = 16476}, rd_smgr = 0x230d3c8, rd_refcnt = 1, rd_backend = -1, 
      rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, rd_indexvalid = 0 '\000', rd_statvalid = false, 
      rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7f6b99fff7f8, rd_att = 0x7f6b9a00f5a0, rd_id = 16476, 
      rd_lockInfo = {lockRelId = {relId = 16476, dbId = 16384}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, 
      rd_rsdesc = 0x0, rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, 
      rd_partdesc = 0x0, rd_partcheck = 0x0, rd_indexlist = 0x0, rd_oidindex = 0, rd_pkindex = 0, rd_replidindex = 0, 
      rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, rd_keyattr = 0x0, rd_pkattr = 0x0, rd_idattr = 0x0, 
      rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x7f6b9a011898, rd_indextuple = 0x7f6b9a011860, 
      rd_amhandler = 330, rd_indexcxt = 0x2313400, rd_amroutine = 0x2313530, rd_opfamily = 0x2313640, rd_opcintype = 0x2313658, 
      rd_support = 0x2313670, rd_supportinfo = 0x2313690, rd_indoption = 0x23137b8, rd_indexprs = 0x0, rd_indpred = 0x0, 
      rd_exclops = 0x0, rd_exclprocs = 0x0, rd_exclstrats = 0x0, rd_amcache = 0x22fada8, rd_indcollation = 0x23137a0, 
      rd_fdwroutine = 0x0, rd_toastoid = 0, pgstat_info = 0x22d1230}
    (gdb) p *indexRelation->rd_rel
    $4 = {relname = {data = "t_dwxx_pkey", '\000' <repeats 52 times>}, relnamespace = 2200, reltype = 0, reloftype = 0, 
      relowner = 10, relam = 403, relfilenode = 16476, reltablespace = 0, relpages = 2, reltuples = 3, relallvisible = 0, 
      reltoastrelid = 0, relhasindex = false, relisshared = false, relpersistence = 112 'p', relkind = 105 'i', relnatts = 1, 
      relchecks = 0, relhasoids = false, relhasrules = false, relhastriggers = false, relhassubclass = false, 
      relrowsecurity = false, relforcerowsecurity = false, relispopulated = true, relreplident = 110 'n', 
      relispartition = false, relrewrite = 0, relfrozenxid = 0, relminmxid = 0}
    ...
    #开始构造IndexOptInfo
    237       info = makeNode(IndexOptInfo);
    (gdb) 
    239       info->indexoid = index->indexrelid;
    (gdb) 
    241         RelationGetForm(indexRelation)->reltablespace;
    (gdb) 
    240       info->reltablespace =
    (gdb) 
    242       info->rel = rel;
    ...
    (gdb) p index->indnatts
    $5 = 1
    (gdb) n
    246       info->indexkeys = (int *) palloc(sizeof(int) * ncolumns);
    (gdb) p index->indnkeyatts
    $6 = 1
    ...
    252       for (i = 0; i < ncolumns; i++)
    (gdb) 
    254         info->indexkeys[i] = index->indkey.values[i];
    (gdb) p index->indkey.values[i]
    $7 = 2
    (gdb) p index->indkey
    $8 = {vl_len_ = 104, ndim = 1, dataoffset = 0, elemtype = 21, dim1 = 1, lbound1 = 0, values = 0x7f6b9a0118c8}
    ...
    ##需结合数据字典查看
    (gdb) p indexRelation->rd_opfamily[i]
    $10 = 1994
    (gdb) p indexRelation->rd_opcintype[i]
    $11 = 25
    (gdb) p indexRelation->rd_indcollation[i]
    $12 = 100
    ...
    ##访问方法,后续做物理优化会使用
    (gdb) p indexRelation->rd_rel->relam
    $13 = 403
    (gdb) p indexRelation->rd_amroutine
    $14 = (struct IndexAmRoutine *) 0x2313530
    (gdb) p *indexRelation->rd_amroutine
    $15 = {type = T_IndexAmRoutine, amstrategies = 5, amsupport = 3, amcanorder = true, amcanorderbyop = false, 
      amcanbackward = true, amcanunique = true, amcanmulticol = true, amoptionalkey = true, amsearcharray = true, 
      amsearchnulls = true, amstorage = false, amclusterable = true, ampredlocks = true, amcanparallel = true, 
      amcaninclude = true, amkeytype = 0, ambuild = 0x4ea341 <btbuild>, ambuildempty = 0x4e282a <btbuildempty>, 
      aminsert = 0x4e28d0 <btinsert>, ambulkdelete = 0x4e37f0 <btbulkdelete>, amvacuumcleanup = 0x4e397f <btvacuumcleanup>, 
      amcanreturn = 0x4e427d <btcanreturn>, amcostestimate = 0x94f0ad <btcostestimate>, amoptions = 0x4e9f7f <btoptions>, 
      amproperty = 0x4e9fa9 <btproperty>, amvalidate = 0x4ecad6 <btvalidate>, ambeginscan = 0x4e2bd8 <btbeginscan>, 
      amrescan = 0x4e2d54 <btrescan>, amgettuple = 0x4e294f <btgettuple>, amgetbitmap = 0x4e2a7f <btgetbitmap>, 
      amendscan = 0x4e2f23 <btendscan>, ammarkpos = 0x4e303c <btmarkpos>, amrestrpos = 0x4e310b <btrestrpos>, 
      amestimateparallelscan = 0x4e3281 <btestimateparallelscan>, aminitparallelscan = 0x4e328c <btinitparallelscan>, 
      amparallelrescan = 0x4e32da <btparallelrescan>}
    ##部分属性值  
    (gdb) p amroutine->amcanorderbyop
    $16 = false
    (gdb) p amroutine->amoptionalkey
    $17 = true
    (gdb) p amroutine->amsearcharray
    $18 = true
    (gdb) p amroutine->amsearchnulls
    $19 = true
    (gdb) p amroutine->amcanparallel
    $20 = true
    ##下面是函数指针
    (gdb) p *amroutine->amgettuple
    $21 = {_Bool (IndexScanDesc, ScanDirection)} 0x4e294f <btgettuple>
    (gdb) p *amroutine->amgetbitmap
    $24 = {int64 (IndexScanDesc, TIDBitmap *)} 0x4e2a7f <btgetbitmap>
    (gdb) p *amroutine->amcostestimate
    $26 = {void (struct PlannerInfo *, struct IndexPath *, double, Cost *, Cost *, Selectivity *, double *, 
        double *)} 0x94f0ad <btcostestimate>
    282       if (info->relam == BTREE_AM_OID)
    ##BTree索引
    ...
    ##PK,唯一性为true
    (gdb) p index->indisunique
    $31 = true
    ...
    (gdb) p info->tree_height
    $32 = 0
    ##第2个索引(函数索引)
    183     foreach(l, indexoidlist)
    (gdb) 
    185       Oid     indexoid = lfirst_oid(l);
    ...
    

进入RelationGetIndexExpressions函数

    
    
    371       info->indexprs = RelationGetIndexExpressions(indexRelation);
    (gdb) step
    RelationGetIndexExpressions (relation=0x7f6b9a011970) at relcache.c:4625
    ##IndexRelation中已有相关信息
    4625    if (relation->rd_indexprs)
    (gdb) n
    4626      return copyObject(relation->rd_indexprs);
    ##函数表达式中的args有相关的参数,类似的分析方法先前已有提及
    (gdb) p *(FuncExpr *)relation->rd_indexprs->head->data.ptr_value
    $36 = {xpr = {type = T_FuncExpr}, funcid = 885, funcresulttype = 25, funcretset = false, funcvariadic = false, 
      funcformat = COERCE_EXPLICIT_CALL, funccollid = 100, inputcollid = 100, args = 0x22fb638, location = -1}
    

回到get_relation_info

    
    
    ...
    ##第3个索引,这是一个部分(条件)索引
    183     foreach(l, indexoidlist)
    (gdb) 
    185       Oid     indexoid = lfirst_oid(l);
    ...
    372       info->indpred = RelationGetIndexPredicate(indexRelation);
    ##这是一个OpExpr,详细的结构先前已有提及
    (gdb) p *(Node *)indexRelation->rd_indpred->head->data.ptr_value
    $38 = {type = T_OpExpr}
    $39 = {xpr = {type = T_OpExpr}, opno = 1209, opfuncid = 850, opresulttype = 16, opretset = false, opcollid = 0, 
      inputcollid = 100, args = 0x23140d8, location = -1}
    ...
    

回到build_simple_rel函数

    
    
    461   if (get_relation_info_hook)
    (gdb) 
    463 }
    (gdb) 
    build_simple_rel (root=0x23107f8, relid=1, parent=0x0) at relnode.c:185
    185       break;
    (gdb) n
    213   root->simple_rel_array[relid] = rel;
    (gdb) 
    221   if (rte->securityQuals)
    (gdb) 
    231   if (rte->inh)
    (gdb) 
    271   return rel;
    (gdb) 
    272 }
    (gdb) 
    

回到add_base_rels_to_query

    
    
    add_base_rels_to_query (root=0x23107f8, jtnode=0x22515f0) at initsplan.c:133
    133 }
    ##递归调用完毕,回到FromExpr->fromlist
    (gdb) n
    add_base_rels_to_query (root=0x23107f8, jtnode=0x2251cc8) at initsplan.c:120
    120     foreach(l, f->fromlist)
    (gdb) n
    133 }
    (gdb) 
    query_planner (root=0x23107f8, tlist=0x2312798, qp_callback=0x76e97d <standard_qp_callback>, qp_extra=0x7ffc7d69a9a0)
        at planmain.c:150
    150   build_base_rel_tlists(root, tlist);
    (gdb) 
    #DONE!
    

### 四、参考资料

[planmain.c](https://doxygen.postgresql.org/planmain_8c_source.html)  
[rel.h](https://doxygen.postgresql.org/rel_8h_source.html)

