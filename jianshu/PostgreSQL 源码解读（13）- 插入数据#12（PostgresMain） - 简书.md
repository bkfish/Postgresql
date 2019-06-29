本文简单介绍了PG插入数据部分的源码，主要内容包括PostgresMain函数的实现逻辑，该函数位于postgres.c文件中。

### 一、源码解读

PostgresMain函数

    
    
    /* ----------------------------------------------------------------     * PostgresMain
     *     postgres main loop -- all backends, interactive or otherwise start here
     *
     * argc/argv are the command line arguments to be used.  (When being forked
     * by the postmaster, these are not the original argv array of the process.)
     * dbname is the name of the database to connect to, or NULL if the database
     * name should be extracted from the command line arguments or defaulted.
     * username is the PostgreSQL user name to be used for the session.
     * ----------------------------------------------------------------     */
    /*
    输入：
        argc/argv-Main函数的输入参数
        dbname-数据库名称
        username-用户名
    输出：
        无
    */
    void
    PostgresMain(int argc, char *argv[],
                 const char *dbname,
                 const char *username)
    {
        int         firstchar;//临时变量，读取输入的Command
        StringInfoData input_message;//字符串增强结构体
        sigjmp_buf  local_sigjmp_buf;//系统变量
        volatile bool send_ready_for_query = true;//
        bool        disable_idle_in_transaction_timeout = false;
    
        /* Initialize startup process environment if necessary. */
        if (!IsUnderPostmaster//未初始化？initialized for the bootstrap/standalone case
            InitStandaloneProcess(argv[0]);//初始化进程
    
        SetProcessingMode(InitProcessing);//设置进程状态为InitProcessing
    
        /*
         * Set default values for command-line options.
         */
        if (!IsUnderPostmaster)
            InitializeGUCOptions();//初始化GUC参数，GUC=Grand Unified Configuration
    
        /*
         * Parse command-line options.
         */
        process_postgres_switches(argc, argv, PGC_POSTMASTER, &dbname);//解析输入参数
    
        /* Must have gotten a database name, or have a default (the username) */
        if (dbname == NULL)//输入的dbname为空
        {
            dbname = username;//设置为用户名
            if (dbname == NULL)//如仍为空，报错
                ereport(FATAL,
                        (errcode(ERRCODE_INVALID_PARAMETER_VALUE),
                         errmsg("%s: no database nor user name specified",
                                progname)));
        }
    
        /* Acquire configuration parameters, unless inherited from postmaster */
        if (!IsUnderPostmaster)
        {
            if (!SelectConfigFiles(userDoption, progname))//读取配置文件conf/hba文件&定位数据目录
                proc_exit(1);
        }
    
        /*
         * Set up signal handlers and masks.
         *
         * Note that postmaster blocked all signals before forking child process,
         * so there is no race condition whereby we might receive a signal before
         * we have set up the handler.
         *
         * Also note: it's best not to use any signals that are SIG_IGNored in the
         * postmaster.  If such a signal arrives before we are able to change the
         * handler to non-SIG_IGN, it'll get dropped.  Instead, make a dummy
         * handler in the postmaster to reserve the signal. (Of course, this isn't
         * an issue for signals that are locally generated, such as SIGALRM and
         * SIGPIPE.)
         */
        if (am_walsender)//wal sender进程？
            WalSndSignals();//如果是，则调用WalSndSignals
        else//不是wal sender进程
        {
            pqsignal(SIGHUP, PostgresSigHupHandler);    /* set flag to read config
                                                         * file */
            pqsignal(SIGINT, StatementCancelHandler);   /* cancel current query */
            pqsignal(SIGTERM, die); /* cancel current query and exit */
    
            /*
             * In a standalone backend, SIGQUIT can be generated from the keyboard
             * easily, while SIGTERM cannot, so we make both signals do die()
             * rather than quickdie().
             */
            if (IsUnderPostmaster)
                pqsignal(SIGQUIT, quickdie);    /* hard crash time */
            else
                pqsignal(SIGQUIT, die); /* cancel current query and exit */
            InitializeTimeouts();   /* establishes SIGALRM handler */
    
            /*
             * Ignore failure to write to frontend. Note: if frontend closes
             * connection, we will notice it and exit cleanly when control next
             * returns to outer loop.  This seems safer than forcing exit in the
             * midst of output during who-knows-what operation...
             */
            pqsignal(SIGPIPE, SIG_IGN);
            pqsignal(SIGUSR1, procsignal_sigusr1_handler);
            pqsignal(SIGUSR2, SIG_IGN);
            pqsignal(SIGFPE, FloatExceptionHandler);
    
            /*
             * Reset some signals that are accepted by postmaster but not by
             * backend
             */
            pqsignal(SIGCHLD, SIG_DFL); /* system() requires this on some
                                         * platforms */
        }
    
        pqinitmask();//Initialize BlockSig, UnBlockSig, and StartupBlockSig.
    
        if (IsUnderPostmaster)
        {
            /* We allow SIGQUIT (quickdie) at all times */
            sigdelset(&BlockSig, SIGQUIT);
        }
    
        PG_SETMASK(&BlockSig);      /* block everything except SIGQUIT */
    
        if (!IsUnderPostmaster)
        {
            /*
             * Validate we have been given a reasonable-looking DataDir (if under
             * postmaster, assume postmaster did this already).
             */
            checkDataDir();//确认数据库路径OK，使用stat命令
    
            /* Change into DataDir (if under postmaster, was done already) */
            ChangeToDataDir();//切换至数据库路径，使用chdir命令
    
            /*
             * Create lockfile for data directory.
             */
            CreateDataDirLockFile(false);//创建锁定文件，CreateLockFile(DIRECTORY_LOCK_FILE, amPostmaster, "", true, DataDir);
    
            /* read control file (error checking and contains config ) */
            LocalProcessControlFile(false);//Read the control file, set respective GUCs.
    
            /* Initialize MaxBackends (if under postmaster, was done already) */
            InitializeMaxBackends();//Initialize MaxBackends value from config options.
        }
    
        /* Early initialization */
        BaseInit();//基础的初始化
    
        /*
         * Create a per-backend PGPROC struct in shared memory, except in the
         * EXEC_BACKEND case where this was done in SubPostmasterMain. We must do
         * this before we can use LWLocks (and in the EXEC_BACKEND case we already
         * had to do some stuff with LWLocks).
         */
    // initialize a per-process data structure for this backend
    #ifdef EXEC_BACKEND
        if (!IsUnderPostmaster)
            InitProcess();
    #else
        InitProcess();
    #endif
    
        /* We need to allow SIGINT, etc during the initial transaction */
        PG_SETMASK(&UnBlockSig);
    
        /*
         * General initialization.
         *
         * NOTE: if you are tempted to add code in this vicinity, consider putting
         * it inside InitPostgres() instead.  In particular, anything that
         * involves database access should be there, not here.
         */
        InitPostgres(dbname, InvalidOid, username, InvalidOid, NULL, false);//Initialize POSTGRES
    
        /*
         * If the PostmasterContext is still around, recycle the space; we don't
         * need it anymore after InitPostgres completes.  Note this does not trash
         * *MyProcPort, because ConnCreate() allocated that space with malloc()
         * ... else we'd need to copy the Port data first.  Also, subsidiary data
         * such as the username isn't lost either; see ProcessStartupPacket().
         */
        if (PostmasterContext)
        {
            MemoryContextDelete(PostmasterContext);
            PostmasterContext = NULL;
        }
    
        SetProcessingMode(NormalProcessing);//完成初始化后，设置进程模式为NormalProcessing
    
        /*
         * Now all GUC states are fully set up.  Report them to client if
         * appropriate.
         */
        BeginReportingGUCOptions();//Report GUC
    
        /*
         * Also set up handler to log session end; we have to wait till now to be
         * sure Log_disconnections has its final value.
         */
        if (IsUnderPostmaster && Log_disconnections)
            on_proc_exit(log_disconnections, 0);//this function adds a callback function to the list of functions invoked by proc_exit()
    
        /* Perform initialization specific to a WAL sender process. */
        if (am_walsender)
            InitWalSender();//初始化 WAL sender process
    
        /*
         * process any libraries that should be preloaded at backend start (this
         * likewise can't be done until GUC settings are complete)
         */
        process_session_preload_libraries();//加载LIB
    
        /*
         * Send this backend's cancellation info to the frontend.
         */
        if (whereToSendOutput == DestRemote)
        {
            StringInfoData buf;
    
            pq_beginmessage(&buf, 'K');
            pq_sendint32(&buf, (int32) MyProcPid);
            pq_sendint32(&buf, (int32) MyCancelKey);
            pq_endmessage(&buf);
            /* Need not flush since ReadyForQuery will do it. */
        }
    
        /* Welcome banner for standalone case */
        if (whereToSendOutput == DestDebug)
            printf("\nPostgreSQL stand-alone backend %s\n", PG_VERSION);
    
        /*
         * Create the memory context we will use in the main loop.
         *
         * MessageContext is reset once per iteration of the main loop, ie, upon
         * completion of processing of each command message from the client.
         */
        //初始化内存上下文：MessageContext
        MessageContext = AllocSetContextCreate(TopMemoryContext,
                                               "MessageContext",
                                               ALLOCSET_DEFAULT_SIZES);
    
        /*
         * Create memory context and buffer used for RowDescription messages. As
         * SendRowDescriptionMessage(), via exec_describe_statement_message(), is
         * frequently executed for ever single statement, we don't want to
         * allocate a separate buffer every time.
         */
        //TODO 传输RowDescription messages？
        row_description_context = AllocSetContextCreate(TopMemoryContext,
                                                        "RowDescriptionContext",
                                                        ALLOCSET_DEFAULT_SIZES);
        MemoryContextSwitchTo(row_description_context);
        initStringInfo(&row_description_buf);
        MemoryContextSwitchTo(TopMemoryContext);
    
        /*
         * Remember stand-alone backend startup time
         */
        if (!IsUnderPostmaster)
            PgStartTime = GetCurrentTimestamp();//记录启动时间
    
        /*
         * POSTGRES main processing loop begins here
         *
         * If an exception is encountered, processing resumes here so we abort the
         * current transaction and start a new one.
         *
         * You might wonder why this isn't coded as an infinite loop around a
         * PG_TRY construct.  The reason is that this is the bottom of the
         * exception stack, and so with PG_TRY there would be no exception handler
         * in force at all during the CATCH part.  By leaving the outermost setjmp
         * always active, we have at least some chance of recovering from an error
         * during error recovery.  (If we get into an infinite loop thereby, it
         * will soon be stopped by overflow of elog.c's internal state stack.)
         *
         * Note that we use sigsetjmp(..., 1), so that this function's signal mask
         * (to wit, UnBlockSig) will be restored when longjmp'ing to here.  This
         * is essential in case we longjmp'd out of a signal handler on a platform
         * where that leaves the signal blocked.  It's not redundant with the
         * unblock in AbortTransaction() because the latter is only called if we
         * were inside a transaction.
         */
    
        if (sigsetjmp(local_sigjmp_buf, 1) != 0)//
        {
            /*
             * NOTE: if you are tempted to add more code in this if-block,
             * consider the high probability that it should be in
             * AbortTransaction() instead.  The only stuff done directly here
             * should be stuff that is guaranteed to apply *only* for outer-level
             * error recovery, such as adjusting the FE/BE protocol status.
             */
    
            /* Since not using PG_TRY, must reset error stack by hand */
            error_context_stack = NULL;
    
            /* Prevent interrupts while cleaning up */
            HOLD_INTERRUPTS();
    
            /*
             * Forget any pending QueryCancel request, since we're returning to
             * the idle loop anyway, and cancel any active timeout requests.  (In
             * future we might want to allow some timeout requests to survive, but
             * at minimum it'd be necessary to do reschedule_timeouts(), in case
             * we got here because of a query cancel interrupting the SIGALRM
             * interrupt handler.)  Note in particular that we must clear the
             * statement and lock timeout indicators, to prevent any future plain
             * query cancels from being misreported as timeouts in case we're
             * forgetting a timeout cancel.
             */
            disable_all_timeouts(false);
            QueryCancelPending = false; /* second to avoid race condition */
            stmt_timeout_active = false;
    
            /* Not reading from the client anymore. */
            DoingCommandRead = false;
    
            /* Make sure libpq is in a good state */
            pq_comm_reset();
    
            /* Report the error to the client and/or server log */
            EmitErrorReport();
    
            /*
             * Make sure debug_query_string gets reset before we possibly clobber
             * the storage it points at.
             */
            debug_query_string = NULL;
    
            /*
             * Abort the current transaction in order to recover.
             */
            AbortCurrentTransaction();
    
            if (am_walsender)
                WalSndErrorCleanup();
    
            PortalErrorCleanup();
            SPICleanup();
    
            /*
             * We can't release replication slots inside AbortTransaction() as we
             * need to be able to start and abort transactions while having a slot
             * acquired. But we never need to hold them across top level errors,
             * so releasing here is fine. There's another cleanup in ProcKill()
             * ensuring we'll correctly cleanup on FATAL errors as well.
             */
            if (MyReplicationSlot != NULL)
                ReplicationSlotRelease();
    
            /* We also want to cleanup temporary slots on error. */
            ReplicationSlotCleanup();
    
            jit_reset_after_error();
    
            /*
             * Now return to normal top-level context and clear ErrorContext for
             * next time.
             */
            MemoryContextSwitchTo(TopMemoryContext);
            FlushErrorState();
    
            /*
             * If we were handling an extended-query-protocol message, initiate
             * skip till next Sync.  This also causes us not to issue
             * ReadyForQuery (until we get Sync).
             */
            if (doing_extended_query_message)
                ignore_till_sync = true;
    
            /* We don't have a transaction command open anymore */
            xact_started = false;
    
            /*
             * If an error occurred while we were reading a message from the
             * client, we have potentially lost track of where the previous
             * message ends and the next one begins.  Even though we have
             * otherwise recovered from the error, we cannot safely read any more
             * messages from the client, so there isn't much we can do with the
             * connection anymore.
             */
            if (pq_is_reading_msg())
                ereport(FATAL,
                        (errcode(ERRCODE_PROTOCOL_VIOLATION),
                         errmsg("terminating connection because protocol synchronization was lost")));
    
            /* Now we can allow interrupts again */
            RESUME_INTERRUPTS();
        }
    
        /* We can now handle ereport(ERROR) */
        PG_exception_stack = &local_sigjmp_buf;
    
        if (!ignore_till_sync)
            send_ready_for_query = true;    /* initially, or after error */
    
        /*
         * Non-error queries loop here.
         */
    
        for (;;)//主循环
        {
            /*
             * At top of loop, reset extended-query-message flag, so that any
             * errors encountered in "idle" state don't provoke skip.
             */
            doing_extended_query_message = false;
    
            /*
             * Release storage left over from prior query cycle, and create a new
             * query input buffer in the cleared MessageContext.
             */
            MemoryContextSwitchTo(MessageContext);//切换至MessageContext
            MemoryContextResetAndDeleteChildren(MessageContext);
    
            initStringInfo(&input_message);//初始化输入的信息
    
            /*
             * Also consider releasing our catalog snapshot if any, so that it's
             * not preventing advance of global xmin while we wait for the client.
             */
            InvalidateCatalogSnapshotConditionally();
    
            /*
             * (1) If we've reached idle state, tell the frontend we're ready for
             * a new query.
             *
             * Note: this includes fflush()'ing the last of the prior output.
             *
             * This is also a good time to send collected statistics to the
             * collector, and to update the PS stats display.  We avoid doing
             * those every time through the message loop because it'd slow down
             * processing of batched messages, and because we don't want to report
             * uncommitted updates (that confuses autovacuum).  The notification
             * processor wants a call too, if we are not in a transaction block.
             */
            if (send_ready_for_query)//I am ready!
            {
                if (IsAbortedTransactionBlockState())
                {
                    set_ps_display("idle in transaction (aborted)", false);
                    pgstat_report_activity(STATE_IDLEINTRANSACTION_ABORTED, NULL);
    
                    /* Start the idle-in-transaction timer */
                    if (IdleInTransactionSessionTimeout > 0)
                    {
                        disable_idle_in_transaction_timeout = true;
                        enable_timeout_after(IDLE_IN_TRANSACTION_SESSION_TIMEOUT,
                                             IdleInTransactionSessionTimeout);
                    }
                }
                else if (IsTransactionOrTransactionBlock())
                {
                    set_ps_display("idle in transaction", false);
                    pgstat_report_activity(STATE_IDLEINTRANSACTION, NULL);
    
                    /* Start the idle-in-transaction timer */
                    if (IdleInTransactionSessionTimeout > 0)
                    {
                        disable_idle_in_transaction_timeout = true;
                        enable_timeout_after(IDLE_IN_TRANSACTION_SESSION_TIMEOUT,
                                             IdleInTransactionSessionTimeout);
                    }
                }
                else
                {
                    ProcessCompletedNotifies();
                    pgstat_report_stat(false);
    
                    set_ps_display("idle", false);
                    pgstat_report_activity(STATE_IDLE, NULL);
                }
    
                ReadyForQuery(whereToSendOutput);
                send_ready_for_query = false;
            }
    
            /*
             * (2) Allow asynchronous signals to be executed immediately if they
             * come in while we are waiting for client input. (This must be
             * conditional since we don't want, say, reads on behalf of COPY FROM
             * STDIN doing the same thing.)
             */
            DoingCommandRead = true;
    
            /*
             * (3) read a command (loop blocks here)
             */
            firstchar = ReadCommand(&input_message);//读取命令
    
            /*
             * (4) disable async signal conditions again.
             *
             * Query cancel is supposed to be a no-op when there is no query in
             * progress, so if a query cancel arrived while we were idle, just
             * reset QueryCancelPending. ProcessInterrupts() has that effect when
             * it's called when DoingCommandRead is set, so check for interrupts
             * before resetting DoingCommandRead.
             */
            CHECK_FOR_INTERRUPTS();
            DoingCommandRead = false;
    
            /*
             * (5) turn off the idle-in-transaction timeout
             */
            if (disable_idle_in_transaction_timeout)
            {
                disable_timeout(IDLE_IN_TRANSACTION_SESSION_TIMEOUT, false);
                disable_idle_in_transaction_timeout = false;
            }
    
            /*
             * (6) check for any other interesting events that happened while we
             * slept.
             */
            if (ConfigReloadPending)
            {
                ConfigReloadPending = false;
                ProcessConfigFile(PGC_SIGHUP);
            }
    
            /*
             * (7) process the command.  But ignore it if we're skipping till
             * Sync.
             */
            if (ignore_till_sync && firstchar != EOF)
                continue;
    
            switch (firstchar)
            {
                case 'Q':           /* simple query */
                    {
                        const char *query_string;
    
                        /* Set statement_timestamp() */
                        SetCurrentStatementStartTimestamp();
    
                        query_string = pq_getmsgstring(&input_message);//SQL语句
                        pq_getmsgend(&input_message);
    
                        if (am_walsender)
                        {
                            if (!exec_replication_command(query_string))
                                exec_simple_query(query_string);
                        }
                        else
                            exec_simple_query(query_string);//执行SQL语句
    
                        send_ready_for_query = true;
                    }
                    break;
    
                case 'P':           /* parse */
                    {
                        const char *stmt_name;
                        const char *query_string;
                        int         numParams;
                        Oid        *paramTypes = NULL;
    
                        forbidden_in_wal_sender(firstchar);
    
                        /* Set statement_timestamp() */
                        SetCurrentStatementStartTimestamp();
    
                        stmt_name = pq_getmsgstring(&input_message);
                        query_string = pq_getmsgstring(&input_message);
                        numParams = pq_getmsgint(&input_message, 2);
                        if (numParams > 0)
                        {
                            int         i;
    
                            paramTypes = (Oid *) palloc(numParams * sizeof(Oid));
                            for (i = 0; i < numParams; i++)
                                paramTypes[i] = pq_getmsgint(&input_message, 4);
                        }
                        pq_getmsgend(&input_message);
    
                        exec_parse_message(query_string, stmt_name,
                                           paramTypes, numParams);
                    }
                    break;
    
                case 'B':           /* bind */
                    forbidden_in_wal_sender(firstchar);
    
                    /* Set statement_timestamp() */
                    SetCurrentStatementStartTimestamp();
    
                    /*
                     * this message is complex enough that it seems best to put
                     * the field extraction out-of-line
                     */
                    exec_bind_message(&input_message);
                    break;
    
                case 'E':           /* execute */
                    {
                        const char *portal_name;
                        int         max_rows;
    
                        forbidden_in_wal_sender(firstchar);
    
                        /* Set statement_timestamp() */
                        SetCurrentStatementStartTimestamp();
    
                        portal_name = pq_getmsgstring(&input_message);
                        max_rows = pq_getmsgint(&input_message, 4);
                        pq_getmsgend(&input_message);
    
                        exec_execute_message(portal_name, max_rows);
                    }
                    break;
    
                case 'F':           /* fastpath function call */
                    forbidden_in_wal_sender(firstchar);
    
                    /* Set statement_timestamp() */
                    SetCurrentStatementStartTimestamp();
    
                    /* Report query to various monitoring facilities. */
                    pgstat_report_activity(STATE_FASTPATH, NULL);
                    set_ps_display("<FASTPATH>", false);
    
                    /* start an xact for this function invocation */
                    start_xact_command();
    
                    /*
                     * Note: we may at this point be inside an aborted
                     * transaction.  We can't throw error for that until we've
                     * finished reading the function-call message, so
                     * HandleFunctionRequest() must check for it after doing so.
                     * Be careful not to do anything that assumes we're inside a
                     * valid transaction here.
                     */
    
                    /* switch back to message context */
                    MemoryContextSwitchTo(MessageContext);
    
                    HandleFunctionRequest(&input_message);
    
                    /* commit the function-invocation transaction */
                    finish_xact_command();
    
                    send_ready_for_query = true;
                    break;
    
                case 'C':           /* close */
                    {
                        int         close_type;
                        const char *close_target;
    
                        forbidden_in_wal_sender(firstchar);
    
                        close_type = pq_getmsgbyte(&input_message);
                        close_target = pq_getmsgstring(&input_message);
                        pq_getmsgend(&input_message);
    
                        switch (close_type)
                        {
                            case 'S':
                                if (close_target[0] != '\0')
                                    DropPreparedStatement(close_target, false);
                                else
                                {
                                    /* special-case the unnamed statement */
                                    drop_unnamed_stmt();
                                }
                                break;
                            case 'P':
                                {
                                    Portal      portal;
    
                                    portal = GetPortalByName(close_target);
                                    if (PortalIsValid(portal))
                                        PortalDrop(portal, false);
                                }
                                break;
                            default:
                                ereport(ERROR,
                                        (errcode(ERRCODE_PROTOCOL_VIOLATION),
                                         errmsg("invalid CLOSE message subtype %d",
                                                close_type)));
                                break;
                        }
    
                        if (whereToSendOutput == DestRemote)
                            pq_putemptymessage('3');    /* CloseComplete */
                    }
                    break;
    
                case 'D':           /* describe */
                    {
                        int         describe_type;
                        const char *describe_target;
    
                        forbidden_in_wal_sender(firstchar);
    
                        /* Set statement_timestamp() (needed for xact) */
                        SetCurrentStatementStartTimestamp();
    
                        describe_type = pq_getmsgbyte(&input_message);
                        describe_target = pq_getmsgstring(&input_message);
                        pq_getmsgend(&input_message);
    
                        switch (describe_type)
                        {
                            case 'S':
                                exec_describe_statement_message(describe_target);
                                break;
                            case 'P':
                                exec_describe_portal_message(describe_target);
                                break;
                            default:
                                ereport(ERROR,
                                        (errcode(ERRCODE_PROTOCOL_VIOLATION),
                                         errmsg("invalid DESCRIBE message subtype %d",
                                                describe_type)));
                                break;
                        }
                    }
                    break;
    
                case 'H':           /* flush */
                    pq_getmsgend(&input_message);
                    if (whereToSendOutput == DestRemote)
                        pq_flush();
                    break;
    
                case 'S':           /* sync */
                    pq_getmsgend(&input_message);
                    finish_xact_command();
                    send_ready_for_query = true;
                    break;
    
                    /*
                     * 'X' means that the frontend is closing down the socket. EOF
                     * means unexpected loss of frontend connection. Either way,
                     * perform normal shutdown.
                     */
                case 'X':
                case EOF:
    
                    /*
                     * Reset whereToSendOutput to prevent ereport from attempting
                     * to send any more messages to client.
                     */
                    if (whereToSendOutput == DestRemote)
                        whereToSendOutput = DestNone;
    
                    /*
                     * NOTE: if you are tempted to add more code here, DON'T!
                     * Whatever you had in mind to do should be set up as an
                     * on_proc_exit or on_shmem_exit callback, instead. Otherwise
                     * it will fail to be called during other backend-shutdown
                     * scenarios.
                     */
                    proc_exit(0);
    
                case 'd':           /* copy data */
                case 'c':           /* copy done */
                case 'f':           /* copy fail */
    
                    /*
                     * Accept but ignore these messages, per protocol spec; we
                     * probably got here because a COPY failed, and the frontend
                     * is still sending data.
                     */
                    break;
    
                default:
                    ereport(FATAL,
                            (errcode(ERRCODE_PROTOCOL_VIOLATION),
                             errmsg("invalid frontend message type %d",
                                    firstchar)));
            }
        }                           /* end of input-reading loop */
    }
    
    

### 二、基础信息

PostgresMain函数使用的数据结构、宏定义以及依赖的函数等。  
**数据结构/宏定义**  
_1、StringInfoData_

    
    
     /*-------------------------      * StringInfoData holds information about an extensible string.
      *      data    is the current buffer for the string (allocated with palloc).
      *      len     is the current string length.  There is guaranteed to be
      *              a terminating '\0' at data[len], although this is not very
      *              useful when the string holds binary data rather than text.
      *      maxlen  is the allocated size in bytes of 'data', i.e. the maximum
      *              string size (including the terminating '\0' char) that we can
      *              currently store in 'data' without having to reallocate
      *              more space.  We must always have maxlen > len.
      *      cursor  is initialized to zero by makeStringInfo or initStringInfo,
      *              but is not otherwise touched by the stringinfo.c routines.
      *              Some routines use it to scan through a StringInfo.
      *-------------------------      */
     typedef struct StringInfoData
     {
         char       *data;
         int         len;
         int         maxlen;
         int         cursor;
     } StringInfoData;
     
     typedef StringInfoData *StringInfo;
    

_2、宏定义_

    
    
      #ifndef WIN32
     #define PG_SETMASK(mask)    sigprocmask(SIG_SETMASK, mask, NULL)
     #else
     /* Emulate POSIX sigset_t APIs on Windows */
     typedef int sigset_t;
     
     extern int  pqsigsetmask(int mask);
     
     #define PG_SETMASK(mask)        pqsigsetmask(*(mask))
     #define sigemptyset(set)        (*(set) = 0)
     #define sigfillset(set)         (*(set) = ~0)
     #define sigaddset(set, signum)  (*(set) |= (sigmask(signum)))
     #define sigdelset(set, signum)  (*(set) &= ~(sigmask(signum)))
     #endif                          /* WIN32 */
    

_3、全局变量_

    
    
    /*
      * IsPostmasterEnvironment is true in a postmaster process and any postmaster
      * child process; it is false in a standalone process (bootstrap or
      * standalone backend).  IsUnderPostmaster is true in postmaster child
      * processes.  Note that "child process" includes all children, not only
      * regular backends.  These should be set correctly as early as possible
      * in the execution of a process, so that error handling will do the right
      * things if an error should occur during process initialization.
      *
      * These are initialized for the bootstrap/standalone case.
      */
     bool        IsPostmasterEnvironment = false;
     bool        IsUnderPostmaster = false;
     bool        IsBinaryUpgrade = false;
     bool IsBackgroundWorker = false;
    
     bool am_walsender = false; /* Am I a walsender process? */
    

**依赖的函数**  
_1、InitStandaloneProcess_

    
    
     /*
      * Initialize the basic environment for a standalone process.
      *
      * argv0 has to be suitable to find the program's executable.
      */
     void
     InitStandaloneProcess(const char *argv0)
     {
         Assert(!IsPostmasterEnvironment);
     
         MyProcPid = getpid();       /* reset MyProcPid */
     
         MyStartTime = time(NULL);   /* set our start time in case we call elog */
     
         /* Initialize process-local latch support */
         InitializeLatchSupport();
         MyLatch = &LocalLatchData;
         InitLatch(MyLatch);
     
         /* Compute paths, no postmaster to inherit from */
         if (my_exec_path[0] == '\0')
         {
             if (find_my_exec(argv0, my_exec_path) < 0)
                 elog(FATAL, "%s: could not locate my own executable path",
                      argv0);
         }
     
         if (pkglib_path[0] == '\0')
             get_pkglib_path(my_exec_path, pkglib_path);
     }
    

_2、InitializeGUCOptions_

    
    
     /*
      * Initialize GUC options during program startup.
      *
      * Note that we cannot read the config file yet, since we have not yet
      * processed command-line switches.
      */
     void
     InitializeGUCOptions(void)
     {
         int         i;
     
         /*
          * Before log_line_prefix could possibly receive a nonempty setting, make
          * sure that timezone processing is minimally alive (see elog.c).
          */
         pg_timezone_initialize();
     
         /*
          * Build sorted array of all GUC variables.
          */
         build_guc_variables();
     
         /*
          * Load all variables with their compiled-in defaults, and initialize
          * status fields as needed.
          */
         for (i = 0; i < num_guc_variables; i++)
         {
             InitializeOneGUCOption(guc_variables[i]);
         }
     
         guc_dirty = false;
     
         reporting_enabled = false;
     
         /*
          * Prevent any attempt to override the transaction modes from
          * non-interactive sources.
          */
         SetConfigOption("transaction_isolation", "default",
                         PGC_POSTMASTER, PGC_S_OVERRIDE);
         SetConfigOption("transaction_read_only", "no",
                         PGC_POSTMASTER, PGC_S_OVERRIDE);
         SetConfigOption("transaction_deferrable", "no",
                         PGC_POSTMASTER, PGC_S_OVERRIDE);
     
         /*
          * For historical reasons, some GUC parameters can receive defaults from
          * environment variables.  Process those settings.
          */
         InitializeGUCOptionsFromEnvironment();
     }
     
    

_3、process_postgres_switches_

    
    
     /* ----------------------------------------------------------------      * process_postgres_switches
      *     Parse command line arguments for PostgresMain
      *
      * This is called twice, once for the "secure" options coming from the
      * postmaster or command line, and once for the "insecure" options coming
      * from the client's startup packet.  The latter have the same syntax but
      * may be restricted in what they can do.
      *
      * argv[0] is ignored in either case (it's assumed to be the program name).
      *
      * ctx is PGC_POSTMASTER for secure options, PGC_BACKEND for insecure options
      * coming from the client, or PGC_SU_BACKEND for insecure options coming from
      * a superuser client.
      *
      * If a database name is present in the command line arguments, it's
      * returned into *dbname (this is allowed only if *dbname is initially NULL).
      * ----------------------------------------------------------------      */
     void
     process_postgres_switches(int argc, char *argv[], GucContext ctx,
                               const char **dbname)
     {
         bool        secure = (ctx == PGC_POSTMASTER);
         int         errs = 0;
         GucSource   gucsource;
         int         flag;
     
         if (secure)
         {
             gucsource = PGC_S_ARGV; /* switches came from command line */
     
             /* Ignore the initial --single argument, if present */
             if (argc > 1 && strcmp(argv[1], "--single") == 0)
             {
                 argv++;
                 argc--;
             }
         }
         else
         {
             gucsource = PGC_S_CLIENT;   /* switches came from client */
         }
     
     #ifdef HAVE_INT_OPTERR
     
         /*
          * Turn this off because it's either printed to stderr and not the log
          * where we'd want it, or argv[0] is now "--single", which would make for
          * a weird error message.  We print our own error message below.
          */
         opterr = 0;
     #endif
     
         /*
          * Parse command-line options.  CAUTION: keep this in sync with
          * postmaster/postmaster.c (the option sets should not conflict) and with
          * the common help() function in main/main.c.
          */
         while ((flag = getopt(argc, argv, "B:bc:C:D:d:EeFf:h:ijk:lN:nOo:Pp:r:S:sTt:v:W:-:")) != -1)
         {
             switch (flag)
             {
                 case 'B':
                     SetConfigOption("shared_buffers", optarg, ctx, gucsource);
                     break;
     
                 case 'b':
                     /* Undocumented flag used for binary upgrades */
                     if (secure)
                         IsBinaryUpgrade = true;
                     break;
     
                 case 'C':
                     /* ignored for consistency with the postmaster */
                     break;
     
                 case 'D':
                     if (secure)
                         userDoption = strdup(optarg);
                     break;
     
                 case 'd':
                     set_debug_options(atoi(optarg), ctx, gucsource);
                     break;
     
                 case 'E':
                     if (secure)
                         EchoQuery = true;
                     break;
     
                 case 'e':
                     SetConfigOption("datestyle", "euro", ctx, gucsource);
                     break;
     
                 case 'F':
                     SetConfigOption("fsync", "false", ctx, gucsource);
                     break;
     
                 case 'f':
                     if (!set_plan_disabling_options(optarg, ctx, gucsource))
                         errs++;
                     break;
     
                 case 'h':
                     SetConfigOption("listen_addresses", optarg, ctx, gucsource);
                     break;
     
                 case 'i':
                     SetConfigOption("listen_addresses", "*", ctx, gucsource);
                     break;
     
                 case 'j':
                     if (secure)
                         UseSemiNewlineNewline = true;
                     break;
     
                 case 'k':
                     SetConfigOption("unix_socket_directories", optarg, ctx, gucsource);
                     break;
     
                 case 'l':
                     SetConfigOption("ssl", "true", ctx, gucsource);
                     break;
     
                 case 'N':
                     SetConfigOption("max_connections", optarg, ctx, gucsource);
                     break;
     
                 case 'n':
                     /* ignored for consistency with postmaster */
                     break;
     
                 case 'O':
                     SetConfigOption("allow_system_table_mods", "true", ctx, gucsource);
                     break;
     
                 case 'o':
                     errs++;
                     break;
     
                 case 'P':
                     SetConfigOption("ignore_system_indexes", "true", ctx, gucsource);
                     break;
     
                 case 'p':
                     SetConfigOption("port", optarg, ctx, gucsource);
                     break;
     
                 case 'r':
                     /* send output (stdout and stderr) to the given file */
                     if (secure)
                         strlcpy(OutputFileName, optarg, MAXPGPATH);
                     break;
     
                 case 'S':
                     SetConfigOption("work_mem", optarg, ctx, gucsource);
                     break;
     
                 case 's':
                     SetConfigOption("log_statement_stats", "true", ctx, gucsource);
                     break;
     
                 case 'T':
                     /* ignored for consistency with the postmaster */
                     break;
     
                 case 't':
                     {
                         const char *tmp = get_stats_option_name(optarg);
     
                         if (tmp)
                             SetConfigOption(tmp, "true", ctx, gucsource);
                         else
                             errs++;
                         break;
                     }
     
                 case 'v':
     
                     /*
                      * -v is no longer used in normal operation, since
                      * FrontendProtocol is already set before we get here. We keep
                      * the switch only for possible use in standalone operation,
                      * in case we ever support using normal FE/BE protocol with a
                      * standalone backend.
                      */
                     if (secure)
                         FrontendProtocol = (ProtocolVersion) atoi(optarg);
                     break;
     
                 case 'W':
                     SetConfigOption("post_auth_delay", optarg, ctx, gucsource);
                     break;
     
                 case 'c':
                 case '-':
                     {
                         char       *name,
                                    *value;
     
                         ParseLongOption(optarg, &name, &value);
                         if (!value)
                         {
                             if (flag == '-')
                                 ereport(ERROR,
                                         (errcode(ERRCODE_SYNTAX_ERROR),
                                          errmsg("--%s requires a value",
                                                 optarg)));
                             else
                                 ereport(ERROR,
                                         (errcode(ERRCODE_SYNTAX_ERROR),
                                          errmsg("-c %s requires a value",
                                                 optarg)));
                         }
                         SetConfigOption(name, value, ctx, gucsource);
                         free(name);
                         if (value)
                             free(value);
                         break;
                     }
     
                 default:
                     errs++;
                     break;
             }
     
             if (errs)
                 break;
         }
     
         /*
          * Optional database name should be there only if *dbname is NULL.
          */
         if (!errs && dbname && *dbname == NULL && argc - optind >= 1)
             *dbname = strdup(argv[optind++]);
     
         if (errs || argc != optind)
         {
             if (errs)
                 optind--;           /* complain about the previous argument */
     
             /* spell the error message a bit differently depending on context */
             if (IsUnderPostmaster)
                 ereport(FATAL,
                         (errcode(ERRCODE_SYNTAX_ERROR),
                          errmsg("invalid command-line argument for server process: %s", argv[optind]),
                          errhint("Try \"%s --help\" for more information.", progname)));
             else
                 ereport(FATAL,
                         (errcode(ERRCODE_SYNTAX_ERROR),
                          errmsg("%s: invalid command-line argument: %s",
                                 progname, argv[optind]),
                          errhint("Try \"%s --help\" for more information.", progname)));
         }
     
         /*
          * Reset getopt(3) library so that it will work correctly in subprocesses
          * or when this function is called a second time with another array.
          */
         optind = 1;
     #ifdef HAVE_INT_OPTRESET
         optreset = 1;               /* some systems need this too */
     #endif
     }
     
    

_4、SelectConfigFiles_

    
    
     /*
      * Select the configuration files and data directory to be used, and
      * do the initial read of postgresql.conf.
      *
      * This is called after processing command-line switches.
      *      userDoption is the -D switch value if any (NULL if unspecified).
      *      progname is just for use in error messages.
      *
      * Returns true on success; on failure, prints a suitable error message
      * to stderr and returns false.
      */
     bool
     SelectConfigFiles(const char *userDoption, const char *progname)
     {
         char       *configdir;
         char       *fname;
         struct stat stat_buf;
     
         /* configdir is -D option, or $PGDATA if no -D */
         if (userDoption)
             configdir = make_absolute_path(userDoption);
         else
             configdir = make_absolute_path(getenv("PGDATA"));
     
         if (configdir && stat(configdir, &stat_buf) != 0)
         {
             write_stderr("%s: could not access directory \"%s\": %s\n",
                          progname,
                          configdir,
                          strerror(errno));
             if (errno == ENOENT)
                 write_stderr("Run initdb or pg_basebackup to initialize a PostgreSQL data directory.\n");
             return false;
         }
     
         /*
          * Find the configuration file: if config_file was specified on the
          * command line, use it, else use configdir/postgresql.conf.  In any case
          * ensure the result is an absolute path, so that it will be interpreted
          * the same way by future backends.
          */
         if (ConfigFileName)
             fname = make_absolute_path(ConfigFileName);
         else if (configdir)
         {
             fname = guc_malloc(FATAL,
                                strlen(configdir) + strlen(CONFIG_FILENAME) + 2);
             sprintf(fname, "%s/%s", configdir, CONFIG_FILENAME);
         }
         else
         {
             write_stderr("%s does not know where to find the server configuration file.\n"
                          "You must specify the --config-file or -D invocation "
                          "option or set the PGDATA environment variable.\n",
                          progname);
             return false;
         }
     
         /*
          * Set the ConfigFileName GUC variable to its final value, ensuring that
          * it can't be overridden later.
          */
         SetConfigOption("config_file", fname, PGC_POSTMASTER, PGC_S_OVERRIDE);
         free(fname);
     
         /*
          * Now read the config file for the first time.
          */
         if (stat(ConfigFileName, &stat_buf) != 0)
         {
             write_stderr("%s: could not access the server configuration file \"%s\": %s\n",
                          progname, ConfigFileName, strerror(errno));
             free(configdir);
             return false;
         }
     
         /*
          * Read the configuration file for the first time.  This time only the
          * data_directory parameter is picked up to determine the data directory,
          * so that we can read the PG_AUTOCONF_FILENAME file next time.
          */
         ProcessConfigFile(PGC_POSTMASTER);
     
         /*
          * If the data_directory GUC variable has been set, use that as DataDir;
          * otherwise use configdir if set; else punt.
          *
          * Note: SetDataDir will copy and absolute-ize its argument, so we don't
          * have to.
          */
         if (data_directory)
             SetDataDir(data_directory);
         else if (configdir)
             SetDataDir(configdir);
         else
         {
             write_stderr("%s does not know where to find the database system data.\n"
                          "This can be specified as \"data_directory\" in \"%s\", "
                          "or by the -D invocation option, or by the "
                          "PGDATA environment variable.\n",
                          progname, ConfigFileName);
             return false;
         }
     
         /*
          * Reflect the final DataDir value back into the data_directory GUC var.
          * (If you are wondering why we don't just make them a single variable,
          * it's because the EXEC_BACKEND case needs DataDir to be transmitted to
          * child backends specially.  XXX is that still true?  Given that we now
          * chdir to DataDir, EXEC_BACKEND can read the config file without knowing
          * DataDir in advance.)
          */
         SetConfigOption("data_directory", DataDir, PGC_POSTMASTER, PGC_S_OVERRIDE);
     
         /*
          * Now read the config file a second time, allowing any settings in the
          * PG_AUTOCONF_FILENAME file to take effect.  (This is pretty ugly, but
          * since we have to determine the DataDir before we can find the autoconf
          * file, the alternatives seem worse.)
          */
         ProcessConfigFile(PGC_POSTMASTER);
     
         /*
          * If timezone_abbreviations wasn't set in the configuration file, install
          * the default value.  We do it this way because we can't safely install a
          * "real" value until my_exec_path is set, which may not have happened
          * when InitializeGUCOptions runs, so the bootstrap default value cannot
          * be the real desired default.
          */
         pg_timezone_abbrev_initialize();
     
         /*
          * Figure out where pg_hba.conf is, and make sure the path is absolute.
          */
         if (HbaFileName)
             fname = make_absolute_path(HbaFileName);
         else if (configdir)
         {
             fname = guc_malloc(FATAL,
                                strlen(configdir) + strlen(HBA_FILENAME) + 2);
             sprintf(fname, "%s/%s", configdir, HBA_FILENAME);
         }
         else
         {
             write_stderr("%s does not know where to find the \"hba\" configuration file.\n"
                          "This can be specified as \"hba_file\" in \"%s\", "
                          "or by the -D invocation option, or by the "
                          "PGDATA environment variable.\n",
                          progname, ConfigFileName);
             return false;
         }
         SetConfigOption("hba_file", fname, PGC_POSTMASTER, PGC_S_OVERRIDE);
         free(fname);
     
         /*
          * Likewise for pg_ident.conf.
          */
         if (IdentFileName)
             fname = make_absolute_path(IdentFileName);
         else if (configdir)
         {
             fname = guc_malloc(FATAL,
                                strlen(configdir) + strlen(IDENT_FILENAME) + 2);
             sprintf(fname, "%s/%s", configdir, IDENT_FILENAME);
         }
         else
         {
             write_stderr("%s does not know where to find the \"ident\" configuration file.\n"
                          "This can be specified as \"ident_file\" in \"%s\", "
                          "or by the -D invocation option, or by the "
                          "PGDATA environment variable.\n",
                          progname, ConfigFileName);
             return false;
         }
         SetConfigOption("ident_file", fname, PGC_POSTMASTER, PGC_S_OVERRIDE);
         free(fname);
     
         free(configdir);
     
         return true;
     }
     
    

_5、pqinitmask_

    
    
     /*
      * Initialize BlockSig, UnBlockSig, and StartupBlockSig.
      *
      * BlockSig is the set of signals to block when we are trying to block
      * signals.  This includes all signals we normally expect to get, but NOT
      * signals that should never be turned off.
      *
      * StartupBlockSig is the set of signals to block during startup packet
      * collection; it's essentially BlockSig minus SIGTERM, SIGQUIT, SIGALRM.
      *
      * UnBlockSig is the set of signals to block when we don't want to block
      * signals (is this ever nonzero??)
      */
     void
     pqinitmask(void)
     {
         sigemptyset(&UnBlockSig);
     
         /* First set all signals, then clear some. */
         sigfillset(&BlockSig);
         sigfillset(&StartupBlockSig);
     
         /*
          * Unmark those signals that should never be blocked. Some of these signal
          * names don't exist on all platforms.  Most do, but might as well ifdef
          * them all for consistency...
          */
     #ifdef SIGTRAP
         sigdelset(&BlockSig, SIGTRAP);
         sigdelset(&StartupBlockSig, SIGTRAP);
     #endif
     #ifdef SIGABRT
         sigdelset(&BlockSig, SIGABRT);
         sigdelset(&StartupBlockSig, SIGABRT);
     #endif
     #ifdef SIGILL
         sigdelset(&BlockSig, SIGILL);
         sigdelset(&StartupBlockSig, SIGILL);
     #endif
     #ifdef SIGFPE
         sigdelset(&BlockSig, SIGFPE);
         sigdelset(&StartupBlockSig, SIGFPE);
     #endif
     #ifdef SIGSEGV
         sigdelset(&BlockSig, SIGSEGV);
         sigdelset(&StartupBlockSig, SIGSEGV);
     #endif
     #ifdef SIGBUS
         sigdelset(&BlockSig, SIGBUS);
         sigdelset(&StartupBlockSig, SIGBUS);
     #endif
     #ifdef SIGSYS
         sigdelset(&BlockSig, SIGSYS);
         sigdelset(&StartupBlockSig, SIGSYS);
     #endif
     #ifdef SIGCONT
         sigdelset(&BlockSig, SIGCONT);
         sigdelset(&StartupBlockSig, SIGCONT);
     #endif
     
     /* Signals unique to startup */
     #ifdef SIGQUIT
         sigdelset(&StartupBlockSig, SIGQUIT);
     #endif
     #ifdef SIGTERM
         sigdelset(&StartupBlockSig, SIGTERM);
     #endif
     #ifdef SIGALRM
         sigdelset(&StartupBlockSig, SIGALRM);
     #endif
     }
    

_6、BaseInit_

    
    
     /*
      * Early initialization of a backend (either standalone or under postmaster).
      * This happens even before InitPostgres.
      *
      * This is separate from InitPostgres because it is also called by auxiliary
      * processes, such as the background writer process, which may not call
      * InitPostgres at all.
      */
     void
     BaseInit(void)
     {
         /*
          * Attach to shared memory and semaphores, and initialize our
          * input/output/debugging file descriptors.
          */
         InitCommunication();
         DebugFileOpen();
     
         /* Do local initialization of file, storage and buffer managers */
         InitFileAccess();
         smgrinit();
         InitBufferPoolAccess();
     }
    

_7、InitProcess_

    
    
     /*
      * InitProcess -- initialize a per-process data structure for this backend
      */
     void
     InitProcess(void)
     {
         PGPROC     *volatile *procgloballist;
     
         /*
          * ProcGlobal should be set up already (if we are a backend, we inherit
          * this by fork() or EXEC_BACKEND mechanism from the postmaster).
          */
         if (ProcGlobal == NULL)
             elog(PANIC, "proc header uninitialized");
     
         if (MyProc != NULL)
             elog(ERROR, "you already exist");
     
         /* Decide which list should supply our PGPROC. */
         if (IsAnyAutoVacuumProcess())
             procgloballist = &ProcGlobal->autovacFreeProcs;
         else if (IsBackgroundWorker)
             procgloballist = &ProcGlobal->bgworkerFreeProcs;
         else
             procgloballist = &ProcGlobal->freeProcs;
     
         /*
          * Try to get a proc struct from the appropriate free list.  If this
          * fails, we must be out of PGPROC structures (not to mention semaphores).
          *
          * While we are holding the ProcStructLock, also copy the current shared
          * estimate of spins_per_delay to local storage.
          */
         SpinLockAcquire(ProcStructLock);
     
         set_spins_per_delay(ProcGlobal->spins_per_delay);
     
         MyProc = *procgloballist;
     
         if (MyProc != NULL)
         {
             *procgloballist = (PGPROC *) MyProc->links.next;
             SpinLockRelease(ProcStructLock);
         }
         else
         {
             /*
              * If we reach here, all the PGPROCs are in use.  This is one of the
              * possible places to detect "too many backends", so give the standard
              * error message.  XXX do we need to give a different failure message
              * in the autovacuum case?
              */
             SpinLockRelease(ProcStructLock);
             ereport(FATAL,
                     (errcode(ERRCODE_TOO_MANY_CONNECTIONS),
                      errmsg("sorry, too many clients already")));
         }
         MyPgXact = &ProcGlobal->allPgXact[MyProc->pgprocno];
     
         /*
          * Cross-check that the PGPROC is of the type we expect; if this were not
          * the case, it would get returned to the wrong list.
          */
         Assert(MyProc->procgloballist == procgloballist);
     
         /*
          * Now that we have a PGPROC, mark ourselves as an active postmaster
          * child; this is so that the postmaster can detect it if we exit without
          * cleaning up.  (XXX autovac launcher currently doesn't participate in
          * this; it probably should.)
          */
         if (IsUnderPostmaster && !IsAutoVacuumLauncherProcess())
             MarkPostmasterChildActive();
     
         /*
          * Initialize all fields of MyProc, except for those previously
          * initialized by InitProcGlobal.
          */
         SHMQueueElemInit(&(MyProc->links));
         MyProc->waitStatus = STATUS_OK;
         MyProc->lxid = InvalidLocalTransactionId;
         MyProc->fpVXIDLock = false;
         MyProc->fpLocalTransactionId = InvalidLocalTransactionId;
         MyPgXact->xid = InvalidTransactionId;
         MyPgXact->xmin = InvalidTransactionId;
         MyProc->pid = MyProcPid;
         /* backendId, databaseId and roleId will be filled in later */
         MyProc->backendId = InvalidBackendId;
         MyProc->databaseId = InvalidOid;
         MyProc->roleId = InvalidOid;
         MyProc->isBackgroundWorker = IsBackgroundWorker;
         MyPgXact->delayChkpt = false;
         MyPgXact->vacuumFlags = 0;
         /* NB -- autovac launcher intentionally does not set IS_AUTOVACUUM */
         if (IsAutoVacuumWorkerProcess())
             MyPgXact->vacuumFlags |= PROC_IS_AUTOVACUUM;
         MyProc->lwWaiting = false;
         MyProc->lwWaitMode = 0;
         MyProc->waitLock = NULL;
         MyProc->waitProcLock = NULL;
     #ifdef USE_ASSERT_CHECKING
         {
             int         i;
     
             /* Last process should have released all locks. */
             for (i = 0; i < NUM_LOCK_PARTITIONS; i++)
                 Assert(SHMQueueEmpty(&(MyProc->myProcLocks[i])));
         }
     #endif
         MyProc->recoveryConflictPending = false;
     
         /* Initialize fields for sync rep */
         MyProc->waitLSN = 0;
         MyProc->syncRepState = SYNC_REP_NOT_WAITING;
         SHMQueueElemInit(&(MyProc->syncRepLinks));
     
         /* Initialize fields for group XID clearing. */
         MyProc->procArrayGroupMember = false;
         MyProc->procArrayGroupMemberXid = InvalidTransactionId;
         pg_atomic_init_u32(&MyProc->procArrayGroupNext, INVALID_PGPROCNO);
     
         /* Check that group locking fields are in a proper initial state. */
         Assert(MyProc->lockGroupLeader == NULL);
         Assert(dlist_is_empty(&MyProc->lockGroupMembers));
     
         /* Initialize wait event information. */
         MyProc->wait_event_info = 0;
     
         /* Initialize fields for group transaction status update. */
         MyProc->clogGroupMember = false;
         MyProc->clogGroupMemberXid = InvalidTransactionId;
         MyProc->clogGroupMemberXidStatus = TRANSACTION_STATUS_IN_PROGRESS;
         MyProc->clogGroupMemberPage = -1;
         MyProc->clogGroupMemberLsn = InvalidXLogRecPtr;
         pg_atomic_init_u32(&MyProc->clogGroupNext, INVALID_PGPROCNO);
     
         /*
          * Acquire ownership of the PGPROC's latch, so that we can use WaitLatch
          * on it.  That allows us to repoint the process latch, which so far
          * points to process local one, to the shared one.
          */
         OwnLatch(&MyProc->procLatch);
         SwitchToSharedLatch();
     
         /*
          * We might be reusing a semaphore that belonged to a failed process. So
          * be careful and reinitialize its value here.  (This is not strictly
          * necessary anymore, but seems like a good idea for cleanliness.)
          */
         PGSemaphoreReset(MyProc->sem);
     
         /*
          * Arrange to clean up at backend exit.
          */
         on_shmem_exit(ProcKill, 0);
     
         /*
          * Now that we have a PGPROC, we could try to acquire locks, so initialize
          * local state needed for LWLocks, and the deadlock checker.
          */
         InitLWLockAccess();
         InitDeadLockChecking();
     }
    

_8、InitPostgres_

    
    
     /* --------------------------------      * InitPostgres
      *      Initialize POSTGRES.
      *
      * The database can be specified by name, using the in_dbname parameter, or by
      * OID, using the dboid parameter.  In the latter case, the actual database
      * name can be returned to the caller in out_dbname.  If out_dbname isn't
      * NULL, it must point to a buffer of size NAMEDATALEN.
      *
      * Similarly, the username can be passed by name, using the username parameter,
      * or by OID using the useroid parameter.
      *
      * In bootstrap mode no parameters are used.  The autovacuum launcher process
      * doesn't use any parameters either, because it only goes far enough to be
      * able to read pg_database; it doesn't connect to any particular database.
      * In walsender mode only username is used.
      *
      * As of PostgreSQL 8.2, we expect InitProcess() was already called, so we
      * already have a PGPROC struct ... but it's not completely filled in yet.
      *
      * Note:
      *      Be very careful with the order of calls in the InitPostgres function.
      * --------------------------------      */
     void
     InitPostgres(const char *in_dbname, Oid dboid, const char *username,
                  Oid useroid, char *out_dbname, bool override_allow_connections)
     {
         bool        bootstrap = IsBootstrapProcessingMode();
         bool        am_superuser;
         char       *fullpath;
         char        dbname[NAMEDATALEN];
     
         elog(DEBUG3, "InitPostgres");
     
         /*
          * Add my PGPROC struct to the ProcArray.
          *
          * Once I have done this, I am visible to other backends!
          */
         InitProcessPhase2();
     
         /*
          * Initialize my entry in the shared-invalidation manager's array of
          * per-backend data.
          *
          * Sets up MyBackendId, a unique backend identifier.
          */
         MyBackendId = InvalidBackendId;
     
         SharedInvalBackendInit(false);
     
         if (MyBackendId > MaxBackends || MyBackendId <= 0)
             elog(FATAL, "bad backend ID: %d", MyBackendId);
     
         /* Now that we have a BackendId, we can participate in ProcSignal */
         ProcSignalInit(MyBackendId);
     
         /*
          * Also set up timeout handlers needed for backend operation.  We need
          * these in every case except bootstrap.
          */
         if (!bootstrap)
         {
             RegisterTimeout(DEADLOCK_TIMEOUT, CheckDeadLockAlert);
             RegisterTimeout(STATEMENT_TIMEOUT, StatementTimeoutHandler);
             RegisterTimeout(LOCK_TIMEOUT, LockTimeoutHandler);
             RegisterTimeout(IDLE_IN_TRANSACTION_SESSION_TIMEOUT,
                             IdleInTransactionSessionTimeoutHandler);
         }
     
         /*
          * bufmgr needs another initialization call too
          */
         InitBufferPoolBackend();
     
         /*
          * Initialize local process's access to XLOG.
          */
         if (IsUnderPostmaster)
         {
             /*
              * The postmaster already started the XLOG machinery, but we need to
              * call InitXLOGAccess(), if the system isn't in hot-standby mode.
              * This is handled by calling RecoveryInProgress and ignoring the
              * result.
              */
             (void) RecoveryInProgress();
         }
         else
         {
             /*
              * We are either a bootstrap process or a standalone backend. Either
              * way, start up the XLOG machinery, and register to have it closed
              * down at exit.
              *
              * We don't yet have an aux-process resource owner, but StartupXLOG
              * and ShutdownXLOG will need one.  Hence, create said resource owner
              * (and register a callback to clean it up after ShutdownXLOG runs).
              */
             CreateAuxProcessResourceOwner();
     
             StartupXLOG();
             /* Release (and warn about) any buffer pins leaked in StartupXLOG */
             ReleaseAuxProcessResources(true);
             /* Reset CurrentResourceOwner to nothing for the moment */
             CurrentResourceOwner = NULL;
     
             on_shmem_exit(ShutdownXLOG, 0);
         }
     
         /*
          * Initialize the relation cache and the system catalog caches.  Note that
          * no catalog access happens here; we only set up the hashtable structure.
          * We must do this before starting a transaction because transaction abort
          * would try to touch these hashtables.
          */
         RelationCacheInitialize();
         InitCatalogCache();
         InitPlanCache();
     
         /* Initialize portal manager */
         EnablePortalManager();
     
         /* Initialize stats collection --- must happen before first xact */
         if (!bootstrap)
             pgstat_initialize();
     
         /*
          * Load relcache entries for the shared system catalogs.  This must create
          * at least entries for pg_database and catalogs used for authentication.
          */
         RelationCacheInitializePhase2();
     
         /*
          * Set up process-exit callback to do pre-shutdown cleanup.  This is the
          * first before_shmem_exit callback we register; thus, this will be the
          * last thing we do before low-level modules like the buffer manager begin
          * to close down.  We need to have this in place before we begin our first
          * transaction --- if we fail during the initialization transaction, as is
          * entirely possible, we need the AbortTransaction call to clean up.
          */
         before_shmem_exit(ShutdownPostgres, 0);
     
         /* The autovacuum launcher is done here */
         if (IsAutoVacuumLauncherProcess())
         {
             /* report this backend in the PgBackendStatus array */
             pgstat_bestart();
     
             return;
         }
     
         /*
          * Start a new transaction here before first access to db, and get a
          * snapshot.  We don't have a use for the snapshot itself, but we're
          * interested in the secondary effect that it sets RecentGlobalXmin. (This
          * is critical for anything that reads heap pages, because HOT may decide
          * to prune them even if the process doesn't attempt to modify any
          * tuples.)
          */
         if (!bootstrap)
         {
             /* statement_timestamp must be set for timeouts to work correctly */
             SetCurrentStatementStartTimestamp();
             StartTransactionCommand();
     
             /*
              * transaction_isolation will have been set to the default by the
              * above.  If the default is "serializable", and we are in hot
              * standby, we will fail if we don't change it to something lower.
              * Fortunately, "read committed" is plenty good enough.
              */
             XactIsoLevel = XACT_READ_COMMITTED;
     
             (void) GetTransactionSnapshot();
         }
     
         /*
          * Perform client authentication if necessary, then figure out our
          * postgres user ID, and see if we are a superuser.
          *
          * In standalone mode and in autovacuum worker processes, we use a fixed
          * ID, otherwise we figure it out from the authenticated user name.
          */
         if (bootstrap || IsAutoVacuumWorkerProcess())
         {
             InitializeSessionUserIdStandalone();
             am_superuser = true;
         }
         else if (!IsUnderPostmaster)
         {
             InitializeSessionUserIdStandalone();
             am_superuser = true;
             if (!ThereIsAtLeastOneRole())
                 ereport(WARNING,
                         (errcode(ERRCODE_UNDEFINED_OBJECT),
                          errmsg("no roles are defined in this database system"),
                          errhint("You should immediately run CREATE USER \"%s\" SUPERUSER;.",
                                  username != NULL ? username : "postgres")));
         }
         else if (IsBackgroundWorker)
         {
             if (username == NULL && !OidIsValid(useroid))
             {
                 InitializeSessionUserIdStandalone();
                 am_superuser = true;
             }
             else
             {
                 InitializeSessionUserId(username, useroid);
                 am_superuser = superuser();
             }
         }
         else
         {
             /* normal multiuser case */
             Assert(MyProcPort != NULL);
             PerformAuthentication(MyProcPort);
             InitializeSessionUserId(username, useroid);
             am_superuser = superuser();
         }
     
         /*
          * If we're trying to shut down, only superusers can connect, and new
          * replication connections are not allowed.
          */
         if ((!am_superuser || am_walsender) &&
             MyProcPort != NULL &&
             MyProcPort->canAcceptConnections == CAC_WAITBACKUP)
         {
             if (am_walsender)
                 ereport(FATAL,
                         (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
                          errmsg("new replication connections are not allowed during database shutdown")));
             else
                 ereport(FATAL,
                         (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
                          errmsg("must be superuser to connect during database shutdown")));
         }
     
         /*
          * Binary upgrades only allowed super-user connections
          */
         if (IsBinaryUpgrade && !am_superuser)
         {
             ereport(FATAL,
                     (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
                      errmsg("must be superuser to connect in binary upgrade mode")));
         }
     
         /*
          * The last few connection slots are reserved for superusers.  Although
          * replication connections currently require superuser privileges, we
          * don't allow them to consume the reserved slots, which are intended for
          * interactive use.
          */
         if ((!am_superuser || am_walsender) &&
             ReservedBackends > 0 &&
             !HaveNFreeProcs(ReservedBackends))
             ereport(FATAL,
                     (errcode(ERRCODE_TOO_MANY_CONNECTIONS),
                      errmsg("remaining connection slots are reserved for non-replication superuser connections")));
     
         /* Check replication permissions needed for walsender processes. */
         if (am_walsender)
         {
             Assert(!bootstrap);
     
             if (!superuser() && !has_rolreplication(GetUserId()))
                 ereport(FATAL,
                         (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
                          errmsg("must be superuser or replication role to start walsender")));
         }
     
         /*
          * If this is a plain walsender only supporting physical replication, we
          * don't want to connect to any particular database. Just finish the
          * backend startup by processing any options from the startup packet, and
          * we're done.
          */
         if (am_walsender && !am_db_walsender)
         {
             /* process any options passed in the startup packet */
             if (MyProcPort != NULL)
                 process_startup_options(MyProcPort, am_superuser);
     
             /* Apply PostAuthDelay as soon as we've read all options */
             if (PostAuthDelay > 0)
                 pg_usleep(PostAuthDelay * 1000000L);
     
             /* initialize client encoding */
             InitializeClientEncoding();
     
             /* report this backend in the PgBackendStatus array */
             pgstat_bestart();
     
             /* close the transaction we started above */
             CommitTransactionCommand();
     
             return;
         }
     
         /*
          * Set up the global variables holding database id and default tablespace.
          * But note we won't actually try to touch the database just yet.
          *
          * We take a shortcut in the bootstrap case, otherwise we have to look up
          * the db's entry in pg_database.
          */
         if (bootstrap)
         {
             MyDatabaseId = TemplateDbOid;
             MyDatabaseTableSpace = DEFAULTTABLESPACE_OID;
         }
         else if (in_dbname != NULL)
         {
             HeapTuple   tuple;
             Form_pg_database dbform;
     
             tuple = GetDatabaseTuple(in_dbname);
             if (!HeapTupleIsValid(tuple))
                 ereport(FATAL,
                         (errcode(ERRCODE_UNDEFINED_DATABASE),
                          errmsg("database \"%s\" does not exist", in_dbname)));
             dbform = (Form_pg_database) GETSTRUCT(tuple);
             MyDatabaseId = HeapTupleGetOid(tuple);
             MyDatabaseTableSpace = dbform->dattablespace;
             /* take database name from the caller, just for paranoia */
             strlcpy(dbname, in_dbname, sizeof(dbname));
         }
         else if (OidIsValid(dboid))
         {
             /* caller specified database by OID */
             HeapTuple   tuple;
             Form_pg_database dbform;
     
             tuple = GetDatabaseTupleByOid(dboid);
             if (!HeapTupleIsValid(tuple))
                 ereport(FATAL,
                         (errcode(ERRCODE_UNDEFINED_DATABASE),
                          errmsg("database %u does not exist", dboid)));
             dbform = (Form_pg_database) GETSTRUCT(tuple);
             MyDatabaseId = HeapTupleGetOid(tuple);
             MyDatabaseTableSpace = dbform->dattablespace;
             Assert(MyDatabaseId == dboid);
             strlcpy(dbname, NameStr(dbform->datname), sizeof(dbname));
             /* pass the database name back to the caller */
             if (out_dbname)
                 strcpy(out_dbname, dbname);
         }
         else
         {
             /*
              * If this is a background worker not bound to any particular
              * database, we're done now.  Everything that follows only makes sense
              * if we are bound to a specific database.  We do need to close the
              * transaction we started before returning.
              */
             if (!bootstrap)
             {
                 pgstat_bestart();
                 CommitTransactionCommand();
             }
             return;
         }
     
         /*
          * Now, take a writer's lock on the database we are trying to connect to.
          * If there is a concurrently running DROP DATABASE on that database, this
          * will block us until it finishes (and has committed its update of
          * pg_database).
          *
          * Note that the lock is not held long, only until the end of this startup
          * transaction.  This is OK since we will advertise our use of the
          * database in the ProcArray before dropping the lock (in fact, that's the
          * next thing to do).  Anyone trying a DROP DATABASE after this point will
          * see us in the array once they have the lock.  Ordering is important for
          * this because we don't want to advertise ourselves as being in this
          * database until we have the lock; otherwise we create what amounts to a
          * deadlock with CountOtherDBBackends().
          *
          * Note: use of RowExclusiveLock here is reasonable because we envision
          * our session as being a concurrent writer of the database.  If we had a
          * way of declaring a session as being guaranteed-read-only, we could use
          * AccessShareLock for such sessions and thereby not conflict against
          * CREATE DATABASE.
          */
         if (!bootstrap)
             LockSharedObject(DatabaseRelationId, MyDatabaseId, 0,
                              RowExclusiveLock);
     
         /*
          * Now we can mark our PGPROC entry with the database ID.
          *
          * We assume this is an atomic store so no lock is needed; though actually
          * things would work fine even if it weren't atomic.  Anyone searching the
          * ProcArray for this database's ID should hold the database lock, so they
          * would not be executing concurrently with this store.  A process looking
          * for another database's ID could in theory see a chance match if it read
          * a partially-updated databaseId value; but as long as all such searches
          * wait and retry, as in CountOtherDBBackends(), they will certainly see
          * the correct value on their next try.
          */
         MyProc->databaseId = MyDatabaseId;
     
         /*
          * We established a catalog snapshot while reading pg_authid and/or
          * pg_database; but until we have set up MyDatabaseId, we won't react to
          * incoming sinval messages for unshared catalogs, so we won't realize it
          * if the snapshot has been invalidated.  Assume it's no good anymore.
          */
         InvalidateCatalogSnapshot();
     
         /*
          * Recheck pg_database to make sure the target database hasn't gone away.
          * If there was a concurrent DROP DATABASE, this ensures we will die
          * cleanly without creating a mess.
          */
         if (!bootstrap)
         {
             HeapTuple   tuple;
     
             tuple = GetDatabaseTuple(dbname);
             if (!HeapTupleIsValid(tuple) ||
                 MyDatabaseId != HeapTupleGetOid(tuple) ||
                 MyDatabaseTableSpace != ((Form_pg_database) GETSTRUCT(tuple))->dattablespace)
                 ereport(FATAL,
                         (errcode(ERRCODE_UNDEFINED_DATABASE),
                          errmsg("database \"%s\" does not exist", dbname),
                          errdetail("It seems to have just been dropped or renamed.")));
         }
     
         /*
          * Now we should be able to access the database directory safely. Verify
          * it's there and looks reasonable.
          */
         fullpath = GetDatabasePath(MyDatabaseId, MyDatabaseTableSpace);
     
         if (!bootstrap)
         {
             if (access(fullpath, F_OK) == -1)
             {
                 if (errno == ENOENT)
                     ereport(FATAL,
                             (errcode(ERRCODE_UNDEFINED_DATABASE),
                              errmsg("database \"%s\" does not exist",
                                     dbname),
                              errdetail("The database subdirectory \"%s\" is missing.",
                                        fullpath)));
                 else
                     ereport(FATAL,
                             (errcode_for_file_access(),
                              errmsg("could not access directory \"%s\": %m",
                                     fullpath)));
             }
     
             ValidatePgVersion(fullpath);
         }
     
         SetDatabasePath(fullpath);
     
         /*
          * It's now possible to do real access to the system catalogs.
          *
          * Load relcache entries for the system catalogs.  This must create at
          * least the minimum set of "nailed-in" cache entries.
          */
         RelationCacheInitializePhase3();
     
         /* set up ACL framework (so CheckMyDatabase can check permissions) */
         initialize_acl();
     
         /*
          * Re-read the pg_database row for our database, check permissions and set
          * up database-specific GUC settings.  We can't do this until all the
          * database-access infrastructure is up.  (Also, it wants to know if the
          * user is a superuser, so the above stuff has to happen first.)
          */
         if (!bootstrap)
             CheckMyDatabase(dbname, am_superuser, override_allow_connections);
     
         /*
          * Now process any command-line switches and any additional GUC variable
          * settings passed in the startup packet.   We couldn't do this before
          * because we didn't know if client is a superuser.
          */
         if (MyProcPort != NULL)
             process_startup_options(MyProcPort, am_superuser);
     
         /* Process pg_db_role_setting options */
         process_settings(MyDatabaseId, GetSessionUserId());
     
         /* Apply PostAuthDelay as soon as we've read all options */
         if (PostAuthDelay > 0)
             pg_usleep(PostAuthDelay * 1000000L);
     
         /*
          * Initialize various default states that can't be set up until we've
          * selected the active user and gotten the right GUC settings.
          */
     
         /* set default namespace search path */
         InitializeSearchPath();
     
         /* initialize client encoding */
         InitializeClientEncoding();
     
         /* Initialize this backend's session state. */
         InitializeSession();
     
         /* report this backend in the PgBackendStatus array */
         if (!bootstrap)
             pgstat_bestart();
     
         /* close the transaction we started above */
         if (!bootstrap)
             CommitTransactionCommand();
     }
     
    

### 三、跟踪分析

插入测试数据：

    
    
    testdb=# -- 获取pid
    testdb=# select pg_backend_pid();
     pg_backend_pid 
    ----------------               1893
    (1 row)
    testdb=# -- 插入1行
    testdb=# insert into t_insert values(23,'I am PostgresMain','I am PostgresMain','I am PostgresMain');
    testdb=# -- 插入1行                         
    insert into t_insert values(23,'I am PostgresMain','I am PostgresMain','I am PostgresMain');
    （挂起）
    

启动gdb，跟踪调试：

    
    
    [root@localhost ~]# gdb -p 1893
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    ...
    #断点设置在循环中
    (gdb) b postgres.c:4013
    Breakpoint 1 at 0x850d26: file postgres.c, line 4013.
    ...
    (gdb) p input_message
    $7 = {data = 0x1508ef0 "insert into t_insert values(23,'I am PostgresMain','I am PostgresMain','I am PostgresMain');", len = 93, maxlen = 1024, cursor = 0}
    (gdb) n
    ...
    4135            switch (firstchar)
    (gdb) 
    4142                        SetCurrentStatementStartTimestamp();
    (gdb) p firstchar
    $8 = 81
    ...
    (gdb) finish
    Run till exit from #0  PostgresMain (argc=1, argv=0x1532aa8, dbname=0x1532990 "testdb", username=0x1532978 "xdb") at postgres.c:4020
    #DONE!
    

使用gdb跟踪postgres进程启动过程：

    
    
    [xdb@localhost ~]$ gdb postgres
    ...
    (gdb) set follow-fork-mode child 
    (gdb) start
    Temporary breakpoint 1 at 0x6f1735: file main.c, line 62.
    Starting program: /appdb/xdb/bin/postgres 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    
    Temporary breakpoint 1, main (argc=1, argv=0x7fffffffe538) at main.c:62
    62      bool        do_check_root = true;
    (gdb) b PostgresMain
    Breakpoint 2 at 0x8507bb: file postgres.c, line 3631.
    (gdb) del 1
    No breakpoint number 1.
    ...
    (gdb) attach 3028
    Attaching to program: /appdb/xdb/bin/postgres, process 3028
    ...
    #连接DB
    [xdb@localhost ~]$ psql -d testdb
    #回到gdb
    #finish 直至进入postmaster.c中的PostgresMain 
    (gdb) finish
    Run till exit from #0  ServerLoop () at postmaster.c:1704
    [New process 3042]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    [Switching to Thread 0x7ffff7feb740 (LWP 3042)]
    
    Breakpoint 2, PostgresMain (argc=1, argv=0xf28ac8, dbname=0xf289b0 "testdb", username=0xf28998 "xdb") at postgres.c:3631
    3631        volatile bool send_ready_for_query = true;
    (gdb) next
    3632        bool        disable_idle_in_transaction_timeout = false;
    (gdb) 
    3635        if (!IsUnderPostmaster)
    (gdb) p IsUnderPostmaster
    $3 = true
    (gdb) p dbname
    $4 = 0xf289b0 "testdb"
    (gdb) p username
    $5 = 0xf28998 "xdb"
    ...
    3845        MessageContext = AllocSetContextCreate(TopMemoryContext,
    (gdb) 
    3855        row_description_context = AllocSetContextCreate(TopMemoryContext,
    (gdb) 
    3858        MemoryContextSwitchTo(row_description_context);
    (gdb) 
    3859        initStringInfo(&row_description_buf);
    (gdb) 
    3860        MemoryContextSwitchTo(TopMemoryContext);
    (gdb) 
    3865        if (!IsUnderPostmaster)
    (gdb) 
    3890        if (sigsetjmp(local_sigjmp_buf, 1) != 0)
    (gdb) p *MessageContext
    $6 = {type = T_AllocSetContext, isReset = true, allowInCritSection = false, methods = 0xb8c720 <AllocSetMethods>, parent = 0xef9b90, firstchild = 0x0, prevchild = 0xf74c80, nextchild = 0xfabb50, 
      name = 0xb4e87c "MessageContext", ident = 0x0, reset_cbs = 0x0}
    ...
    (gdb) n
    4090            DoingCommandRead = true;
    (gdb) 
    4095            firstchar = ReadCommand(&input_message);
    (gdb) 
    
    4106            CHECK_FOR_INTERRUPTS();
    (gdb) 
    4107            DoingCommandRead = false;
    (gdb) p firstchar
    $8 = 81
    (gdb) p input_message
    $9 = {data = 0xeff010 "insert into t_insert values(24,'I am PostgresMain','I am PostgresMain','I am PostgresMain');", len = 93, maxlen = 1024, cursor = 0}
    (gdb) n
    ...
    #DONE!
    

### 四、小结

1、数据结构：MessageContext ：Top消息内存上下文；  
2、处理/数据流程：进程初始化的相关逻辑。

