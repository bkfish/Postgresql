本文简单介绍了PG插入数据部分的源码，主要内容包括ExecProcNode和ExecProcNodeFirst函数的实现逻辑，ExecProcNode函数位于executor.h文件中，ExecProcNodeFirst函数位于execProcnode.c文件中。

### 一、基础信息

ExecProcNode/ExecProcNodeFirst函数使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**  
_1、ExecProcNodeMtd_  
_ExecProcNodeMtd是一个函数指针类型，指向的函数输入参数是PlanState结构体指针，输出参数是TupleTableSlot 结构体指针_

    
    
     /* ----------------      *   ExecProcNodeMtd
      *
      * This is the method called by ExecProcNode to return the next tuple
      * from an executor node.  It returns NULL, or an empty TupleTableSlot,
      * if no more tuples are available.
      * ----------------      */
     typedef TupleTableSlot *(*ExecProcNodeMtd) (struct PlanState *pstate);
    

**依赖的函数**  
_1、check_stack_depth_

    
    
    //检查stack的深度，如超出系统限制，则主动报错
     /*
      * check_stack_depth/stack_is_too_deep: check for excessively deep recursion
      *
      * This should be called someplace in any recursive routine that might possibly
      * recurse deep enough to overflow the stack.  Most Unixen treat stack
      * overflow as an unrecoverable SIGSEGV, so we want to error out ourselves
      * before hitting the hardware limit.
      *
      * check_stack_depth() just throws an error summarily.  stack_is_too_deep()
      * can be used by code that wants to handle the error condition itself.
      */
     void
     check_stack_depth(void)
     {
         if (stack_is_too_deep())
         {
             ereport(ERROR,
                     (errcode(ERRCODE_STATEMENT_TOO_COMPLEX),
                      errmsg("stack depth limit exceeded"),
                      errhint("Increase the configuration parameter \"max_stack_depth\" (currently %dkB), "
                              "after ensuring the platform's stack depth limit is adequate.",
                              max_stack_depth)));
         }
     }
     
     bool
     stack_is_too_deep(void)
     {
         char        stack_top_loc;
         long        stack_depth;
     
         /*
          * Compute distance from reference point to my local variables
          */
         stack_depth = (long) (stack_base_ptr - &stack_top_loc);
     
         /*
          * Take abs value, since stacks grow up on some machines, down on others
          */
         if (stack_depth < 0)
             stack_depth = -stack_depth;
     
         /*
          * Trouble?
          *
          * The test on stack_base_ptr prevents us from erroring out if called
          * during process setup or in a non-backend process.  Logically it should
          * be done first, but putting it here avoids wasting cycles during normal
          * cases.
          */
         if (stack_depth > max_stack_depth_bytes &&
             stack_base_ptr != NULL)
             return true;
     
         /*
          * On IA64 there is a separate "register" stack that requires its own
          * independent check.  For this, we have to measure the change in the
          * "BSP" pointer from PostgresMain to here.  Logic is just as above,
          * except that we know IA64's register stack grows up.
          *
          * Note we assume that the same max_stack_depth applies to both stacks.
          */
     #if defined(__ia64__) || defined(__ia64)
         stack_depth = (long) (ia64_get_bsp() - register_stack_base_ptr);
     
         if (stack_depth > max_stack_depth_bytes &&
             register_stack_base_ptr != NULL)
             return true;
     #endif                          /* IA64 */
     
         return false;
     }
    

_2、ExecProcNodeInstr_

    
    
     /*
      * ExecProcNode wrapper that performs instrumentation calls.  By keeping
      * this a separate function, we avoid overhead in the normal case where
      * no instrumentation is wanted.
      */
     static TupleTableSlot *
     ExecProcNodeInstr(PlanState *node)
     {
         TupleTableSlot *result;
     
         InstrStartNode(node->instrument);
     
         result = node->ExecProcNodeReal(node);
     
         InstrStopNode(node->instrument, TupIsNull(result) ? 0.0 : 1.0);
     
         return result;
     }
    

### 二、源码解读

_1、ExecProcNode_

    
    
    //外部调用者可通过改变node实现遍历
     /* ----------------------------------------------------------------      *      ExecProcNode
      *
      *      Execute the given node to return a(nother) tuple.
      * ----------------------------------------------------------------      */
     #ifndef FRONTEND
     static inline TupleTableSlot *
     ExecProcNode(PlanState *node)
     {
         if (node->chgParam != NULL) /* something changed? */
             ExecReScan(node);       /* let ReScan handle this */
     
         return node->ExecProcNode(node);
     }
     #endif
     
    

_2、ExecProcNodeFirst_

    
    
    /*
     * ExecProcNode wrapper that performs some one-time checks, before calling
     * the relevant node method (possibly via an instrumentation wrapper).
     */
    /*
    输入：
        node-PlanState指针
    输出：
        存储Tuple的Slot
    */
    static TupleTableSlot *
    ExecProcNodeFirst(PlanState *node)
    {
        /*
         * Perform stack depth check during the first execution of the node.  We
         * only do so the first time round because it turns out to not be cheap on
         * some common architectures (eg. x86).  This relies on the assumption
         * that ExecProcNode calls for a given plan node will always be made at
         * roughly the same stack depth.
         */
        //检查Stack是否超深
        check_stack_depth();
    
        /*
         * If instrumentation is required, change the wrapper to one that just
         * does instrumentation.  Otherwise we can dispense with all wrappers and
         * have ExecProcNode() directly call the relevant function from now on.
         */
        //如果instrument（TODO）
        if (node->instrument)
            node->ExecProcNode = ExecProcNodeInstr;
        else
            node->ExecProcNode = node->ExecProcNodeReal;
        //执行该Node的处理过程
        return node->ExecProcNode(node);
    }
    
    

### 三、跟踪分析

插入测试数据：

    
    
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               2835
    (1 row)
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(14,'ExecProcNodeFirst','ExecProcNodeFirst','ExecProcNodeFirst');
    (挂起)
    
    

启动gdb分析：

    
    
    [root@localhost ~]# gdb -p 2835
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    (gdb) b ExecProcNodeFirst
    Breakpoint 1 at 0x69a797: file execProcnode.c, line 433.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecProcNodeFirst (node=0x2cca790) at execProcnode.c:433
    433     check_stack_depth();
    #查看输入参数
    (gdb) p *node
    $1 = {type = T_ModifyTableState, plan = 0x2c1d028, state = 0x2cca440, ExecProcNode = 0x69a78b <ExecProcNodeFirst>, ExecProcNodeReal = 0x6c2485 <ExecModifyTable>, instrument = 0x0, 
      worker_instrument = 0x0, qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2ccb6a0, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, 
      scandesc = 0x0}
    #ExecProcNode 实际对应的函数是ExecProcNodeFirst
    #ExecProcNodeReal 实际对应的函数是ExecModifyTable（上一章节已粗略解析）
    (gdb) next
    440     if (node->instrument)
    (gdb) 
    #实际调用ExecModifyTable函数（这个函数由更高层的调用函数植入）
    443         node->ExecProcNode = node->ExecProcNodeReal;
    (gdb) 
    445     return node->ExecProcNode(node);
    (gdb) next
    #第二次调用（TODO）
    Breakpoint 1, ExecProcNodeFirst (node=0x2ccac80) at execProcnode.c:433
    433     check_stack_depth();
    (gdb) next
    440     if (node->instrument)
    (gdb) next
    443         node->ExecProcNode = node->ExecProcNodeReal;
    (gdb) next
    445     return node->ExecProcNode(node);
    (gdb) next
    446 }
    (gdb) next
    ExecProcNode (node=0x2ccac80) at ../../../src/include/executor/executor.h:238
    238 }
    #第二次调用的参数
    (gdb) p *node
    $2 = {type = T_ResultState, plan = 0x2cd0488, state = 0x2cca440, ExecProcNode = 0x6c5094 <ExecResult>, ExecProcNodeReal = 0x6c5094 <ExecResult>, instrument = 0x0, worker_instrument = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x2ccad90, ps_ExprContext = 0x2ccab30, ps_ProjInfo = 0x2ccabc0, scandesc = 0x0}
    #ExecProcNode对应的实际函数是ExecResult
    (gdb) 
    

### 四、小结

1、C语言中的多态：C语言中使用函数指针实现了函数的“多态”，增强了代码的可重用性，当然，代价是复杂度有所提升；  
2、参数构造：更详细的参数构造，需要上层调用的进一步解读。

