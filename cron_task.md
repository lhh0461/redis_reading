#服务器定时器
    服务器的高性能定时器一般有两种
    一种是高精确的时间堆定时器，每个定时任务按照最小堆的方式排好序，每次超时必然是堆顶的元素，每次系统触发超时把顶端元素清除，然后执行一次上滤操作。
    一种是精度没那么高的时间轮，时间轮有一个心跳间隔，所有的定时任务的超时时间都是这个心跳间隔的倍数，时间轮类似水表有多个精度不一的时间轮片，当增加一个定时任务时，会算出定时任务所属的轮片，添加到该轮片中，每当小精度的轮片跑完一圈了，它下一阶段的轮片就会相应跳一格。
    但是redis都没用到这两种定时器，它只用到了一个未经排序的列表来存储所有的定时器任务，为何如此？因为redis目前只有一个定时任务，所以就算用普通列表来存储也就问题不大。

#redis定时器
    //redis开服的时候会自动加一个定时任务，每一毫秒执行serverCron这个函数
    if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        serverPanic("Can't create event loop timers.");
        exit(1);
    }

    //创建时间事件的接口实现就是分配一个时间事件结构体，然后填充超时时机，然后添加到定时器列表头。
    long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
            aeTimeProc *proc, void *clientData,
                    aeEventFinalizerProc *finalizerProc)
    {
        long long id = eventLoop->timeEventNextId++;
        aeTimeEvent *te;
    
        te = zmalloc(sizeof(*te));
        if (te == NULL) return AE_ERR;
        te->id = id;
        aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
        te->timeProc = proc;
        te->finalizerProc = finalizerProc;
        te->clientData = clientData;
        te->next = eventLoop->timeEventHead;
        eventLoop->timeEventHead = te;
        return id;
    }

    /* 这个接口就是服务器的定时器处理函数 */
    int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData)
    {
    
        /* Software watchdog: deliver the SIGALRM that will reach the signal
         * handler if we don't return here fast enough. */
        if (server.watchdog_period) watchdogScheduleSignal(server.watchdog_period);
    
        /* 更新缓存的时间，缓存了时间就不用每次调用系统调用获取时间 */
        updateCachedTime();
    
        /* We have just LRU_BITS bits per object for LRU information.
         * So we use an (eventually wrapping) LRU clock.
         *
         * Note that even if the counter wraps it's not a big problem,
         * everything will still work but some object will appear younger
         * to Redis. However for this to happen a given object should never be
         * touched for all the time needed to the counter to wrap, which is
         * not likely.
         *
         * Note that you can change the resolution altering the
         * LRU_CLOCK_RESOLUTION define. */
        unsigned long lruclock = getLRUClock();
        atomicSet(server.lruclock,lruclock);
    
        /* 记录内存使用峰值 */
        if (zmalloc_used_memory() > server.stat_peak_memory)
            server.stat_peak_memory = zmalloc_used_memory();
    
        /* Sample the RSS here since this is a relatively slow call. */
        server.resident_set_size = zmalloc_get_rss();
    
        //当我们收到SIGTERM信号，直接在信号处理函数直接准备关服不安全，异步来处理关服的落盘
        //问题：为什么是不安全的？
        if (server.shutdown_asap) {
            //这个接口写rdb文件
            if (prepareForShutdown(SHUTDOWN_NOFLAGS) == C_OK) exit(0);
            serverLog(LL_WARNING,"SIGTERM received but errors trying to shut down the server, check the logs for more information");
            server.shutdown_asap = 0;
        }
    
        /* 每5秒打印一次数据库信息 */
        run_with_period(5000) {
            for (j = 0; j < server.dbnum; j++) {
                long long size, used, vkeys;
    
                size = dictSlots(server.db[j].dict);
                used = dictSize(server.db[j].dict);
                vkeys = dictSize(server.db[j].expires);
                if (used || vkeys) {
                    serverLog(LL_VERBOSE,"DB %d: %lld keys (%lld volatile) in %lld slots HT.",j,used,vkeys,size);
                    /* dictPrintStats(server.dict); */
                }
            }
        }
    
        /* 每5秒打印一次客户端连接信息 */
        if (!server.sentinel_mode) {
            run_with_period(5000) {
                serverLog(LL_VERBOSE,
                        "%lu clients connected (%lu slaves), %zu bytes in use",
                        listLength(server.clients)-listLength(server.slaves),
                        listLength(server.slaves),
                        zmalloc_used_memory());
            }
        }
    
        /* We need to do a few operations on clients asynchronously. */
        clientsCron();
    
        /* Handle background operations on Redis databases. */
        databasesCron();
    
        /* Start a scheduled AOF rewrite if this was requested by the user while
         * a BGSAVE was in progress. */
        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
                server.aof_rewrite_scheduled)
        {
            rewriteAppendOnlyFileBackground();
        }
    
        /* Check if a background saving or AOF rewrite in progress terminated. */
        if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
                ldbPendingChildren())
        {
            int statloc;
            pid_t pid;
    
            if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
                int exitcode = WEXITSTATUS(statloc);
                int bysignal = 0;
    
                if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);
    
                if (pid == -1) {
                    serverLog(LL_WARNING,"wait3() returned an error: %s. "
                            "rdb_child_pid = %d, aof_child_pid = %d",
                            strerror(errno),
                            (int) server.rdb_child_pid,
                            (int) server.aof_child_pid);
                } else if (pid == server.rdb_child_pid) {
                    backgroundSaveDoneHandler(exitcode,bysignal);
                    if (!bysignal && exitcode == 0) receiveChildInfo();
                } else if (pid == server.aof_child_pid) {
                    backgroundRewriteDoneHandler(exitcode,bysignal);
                    if (!bysignal && exitcode == 0) receiveChildInfo();
                } else {
                    if (!ldbRemoveChild(pid)) {
                        serverLog(LL_WARNING,
                                "Warning, detected child with unmatched pid: %ld",
                                (long)pid);
                    }
                }
                updateDictResizePolicy();
                closeChildInfoPipe();
            }
        } else {
            /* If there is not a background saving/rewrite in progress check if
             * we have to save/rewrite now. */
            for (j = 0; j < server.saveparamslen; j++) {
                struct saveparam *sp = server.saveparams+j;
    
                /* Save if we reached the given amount of changes,
                 * the given amount of seconds, and if the latest bgsave was
                 * successful or if, in case of an error, at least
                 * CONFIG_BGSAVE_RETRY_DELAY seconds already elapsed. */
                if (server.dirty >= sp->changes &&
                        server.unixtime-server.lastsave > sp->seconds &&
                        (server.unixtime-server.lastbgsave_try >
                         CONFIG_BGSAVE_RETRY_DELAY ||
                         server.lastbgsave_status == C_OK))
                {
                    serverLog(LL_NOTICE,"%d changes in %d seconds. Saving...",
                            sp->changes, (int)sp->seconds);
                    rdbSaveInfo rsi, *rsiptr;
                    rsiptr = rdbPopulateSaveInfo(&rsi);
                    rdbSaveBackground(server.rdb_filename,rsiptr);
                    break;
                }
            }
    
            /* Trigger an AOF rewrite if needed. */
            if (server.aof_state == AOF_ON &&
                    server.rdb_child_pid == -1 &&
                    server.aof_child_pid == -1 &&
                    server.aof_rewrite_perc &&
                    server.aof_current_size > server.aof_rewrite_min_size)
            {
                long long base = server.aof_rewrite_base_size ?
                    server.aof_rewrite_base_size : 1;
                long long growth = (server.aof_current_size*100/base) - 100;
                if (growth >= server.aof_rewrite_perc) {
                    serverLog(LL_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                    rewriteAppendOnlyFileBackground();
                }
            }
        }
    
    
        /* AOF postponed flush: Try at every cron cycle if the slow fsync
         * completed. */
        if (server.aof_flush_postponed_start) flushAppendOnlyFile(0);
    
        /* AOF write errors: in this case we have a buffer to flush as well and
         * clear the AOF error in case of success to make the DB writable again,
         * however to try every second is enough in case of 'hz' is set to
         * an higher frequency. */
        run_with_period(1000) {
            if (server.aof_last_write_status == C_ERR)
                flushAppendOnlyFile(0);
        }
    
        /* Close clients that need to be closed asynchronous */
        freeClientsInAsyncFreeQueue();
    
        /* Clear the paused clients flag if needed. */
        clientsArePaused(); /* Don't check return value, just use the side effect.*/
    
        /* Replication cron function -- used to reconnect to master,
         * detect transfer failures, start background RDB transfers and so forth. */
        run_with_period(1000) replicationCron();
    
        /* Run the Redis Cluster cron. */
        run_with_period(100) {
            if (server.cluster_enabled) clusterCron();
        }
    
        /* Run the Sentinel timer if we are in sentinel mode. */
        run_with_period(100) {
            if (server.sentinel_mode) sentinelTimer();
        }
    
        /* Cleanup expired MIGRATE cached sockets. */
        run_with_period(1000) {
            migrateCloseTimedoutSockets();
        }
    
        /* Start a scheduled BGSAVE if the corresponding flag is set. This is
         * useful when we are forced to postpone a BGSAVE because an AOF
         * rewrite is in progress.
         *
         * Note: this code must be after the replicationCron() call above so
         * make sure when refactoring this file to keep this order. This is useful
         * because we want to give priority to RDB savings for replication. */
        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
                server.rdb_bgsave_scheduled &&
                (server.unixtime-server.lastbgsave_try > CONFIG_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == C_OK))
        {
            rdbSaveInfo rsi, *rsiptr;
            rsiptr = rdbPopulateSaveInfo(&rsi);
            if (rdbSaveBackground(server.rdb_filename,rsiptr) == C_OK)
                server.rdb_bgsave_scheduled = 0;
        }
    
        server.cronloops++;
        return 1000/server.hz;
    }

    //处理客户端相关的定时任务
    void clientsCron(void) {
        int numclients = listLength(server.clients);
        //确保每次循环检查的客户端数为总数/server.hz，而这个接口每秒会调用hz次，所以
        //一秒钟内肯会遍历完所有的客户端
        int iterations = numclients/server.hz;
        mstime_t now = mstime();
    
        //如果numclients小于server.hz，那么iterations等于0，但也要处理一些客户端
        if (iterations < CLIENTS_CRON_MIN_ITERATIONS)
            iterations = (numclients < CLIENTS_CRON_MIN_ITERATIONS) ?
                numclients : CLIENTS_CRON_MIN_ITERATIONS;
 
        while(listLength(server.clients) && iterations--) {
            client *c;
            listNode *head;
    
            /* Rotate the list, take the current head, process.
             * This way if the client must be removed from the list it's the
             * first element and we don't incur into O(N) computation. */
            //这个接口是把列表的尾指针挪到列表头
            listRotate(server.clients);
            head = listFirst(server.clients);
            c = listNodeValue(head);
            /* The following functions do different service checks on the client.
             * The protocol is that they return non-zero if the client was
             * terminated. */
            //检查这个客户端是否是超时，超时的话就清除这个客户端
            if (clientsCronHandleTimeout(c,now)) continue;
            //检查是否需要回收额外的查询buffer
            if (clientsCronResizeQueryBuffer(c)) continue;
        }
    }
    

    /* This function handles 'background' operations we are required to do
     * incrementally in Redis databases, such as active key expiring, resizing,
      * rehashing. */
    //这个接口处理后台操作和数据库相关的，像主动的key过期检测，重新分配，重新哈希
    void databasesCron(void) {
        /* Expire keys by random sampling. Not required for slaves
         * as master will synthesize DELs for us. */
        if (server.active_expire_enabled && server.masterhost == NULL) {
            activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
        } else if (server.masterhost != NULL) {
            expireSlaveKeys();
        }
    
        /* Defrag keys gradually. */
        if (server.active_defrag_enabled)
            activeDefragCycle();
    
        /* Perform hash tables rehashing if needed, but only if there are no
         * other processes saving the DB on disk. Otherwise rehashing is bad
         * as will cause a lot of copy-on-write of memory pages. */
        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1) {
            /* We use global counters so if we stop the computation at a given
             * DB we'll be able to start from the successive in the next
             * cron loop iteration. */
            static unsigned int resize_db = 0;
            static unsigned int rehash_db = 0;
            int dbs_per_call = CRON_DBS_PER_CALL;
            int j;
    
            /* Don't test more DBs than we have. */
            if (dbs_per_call > server.dbnum) dbs_per_call = server.dbnum;
    
            /* Resize */
            for (j = 0; j < dbs_per_call; j++) {
                tryResizeHashTables(resize_db % server.dbnum);
                resize_db++;
            }
    
            /* Rehash */
            if (server.activerehashing) {
                for (j = 0; j < dbs_per_call; j++) {
                    int work_done = incrementallyRehash(rehash_db);
                    if (work_done) {
                        /* If the function did some work, stop here, we'll do
                         * more at the next cron loop. */
                        break;
                    } else {
                        /* If this db didn't need rehash, we'll try the next one. */
                        rehash_db++;
                        rehash_db %= server.dbnum;
                    }
                }
            }
        }
    }

