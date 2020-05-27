### Master Thread
####    Structure
            拥有最高的线程优先级别, 由多个loop组成, master thread 会根据数据库运行状态再以下loop中切换:
            1. 主循环(loop)
                多数操作都在这个循环中, 其中有两部分操作[1s, 10s], 伪代码:
                def master_thread()
                    loop:
                    for in range(10)
                        // do things once per second
                        sleep 1 second if necessary
                    // do things once per 10 sec
                    goto loop
                
                loop循环通过thread sleep来实现, 这意味这所谓1秒1次或者10秒一次的操作并不准确, 再负载的的情况小会有延迟, 只能说大概再这个频率下, 当然InnoDB源码中还通过了其他方法来尽量保证这个频率:
                    1秒一次的操作包括:
                        a. 日志缓冲刷新到硬盘, 即使这个事务还没有提交(always)
                            即使某个事务还没有提交, InnoDB依然会每秒将redo log buffer写入redo log, 这就是为什么再大的事务提交(commit)的时间也很短
                        b. 合并插入缓冲(insert buffer)(sometimes)
                            insert buffer并不是每秒都会发生, InnoDB会判断当前1秒内发生的IO次数是否小于5次, 如果小于5次, InnoDB认为当前的IO压力很小, 可以执行合并insert buffer
                        c. 之多写入100个InnoDB的缓冲池中的dirty page到硬盘(sometimes)
                            写入100个dirty page也不是每秒都会发生, InnoDB通过判断当前缓冲池中的脏页比例(buf_get_modified_ratio_pct)是否超过配置文件中的innodb_max_dirty_pages_pct这个参数, 如果超过, InnoDB认为需要做写入硬盘操作, 将100个dirty page写入硬盘
                        d. 如果当前没有用户活动, 则切换到background loop(sometimes)
                
                增加以上条件后, 伪代码:
                def master_thread()
                    loop:
                    for in range(10):
                        thread_sleep(1)    // sleep 1 sec
                        log_buffer_flush() // flush log buffer to disk
                        if last_one_sec_IO < 5:
                            merge_insert_buffer() // merge at most 5 insert buffer
                        if buf_get_modified_ratio_pct > innodb_max_dirty_page_pct :
                            buffer_pool_flush_100_dirty_page()
                        if no user activity:
                            goto background_loop:
                    // do things once per 10 sec
                    background_loop:
                        do things
                        goto loop
                    
                    10秒一次的操作:
                        a. 写入100个dirty page到硬盘(可能的情况下)
                            InnoDB先回判断过去10秒内硬盘IO操作是否小于200次, 如果是, InnoDB认为当前有足够硬盘IO操作能力, 因此将100个dirty page写入硬盘
                        b. merge at most 5 insert buffer(always)
                            merge insert buffer always happens
                        c. 将日志缓冲刷新到磁盘(always)
                            日志缓冲一定会刷新到硬盘(和每秒一样)
                        d. 删除无用的Undo page(always)
                            InnoDB会进行一步执行Full purge操作, 即删除无用的Undo page, 对表进行update, delete这类操作时, 原先的行被标记为删除, 但是因为一致性读(consistent read)的关系, 需要保留这些行版本信息, 但是在full purge过程中, InnoDB会判断当前事务系统中已经被删除的行是否可以删除, 比如有时候可能还有查询需要读之前的undo信息, 如果可以删除, InnoDB会立即将其删除. 源码中, InnoDB在执行full purge是, 每次最多尝试回收20个undo page
                        e. 写入100个或者10个dirty page到硬盘(总是)
                            InnoDB会判断缓冲中dirty page的比例(buf_get_modified_ratio_pct), 如果有超过70%的dirty page, 则刷新100个脏页到硬盘, 如果dirty page比例小于70%, 则刷新10个(*原书写成10%)dirty page到硬盘
                增加条件后, 伪代码:
                def master_thread()
                    loop:
                    for in range(10):
                        thread_sleep(1)    // sleep 1 sec
                        log_buffer_flush() // flush log buffer to disk
                        if last_one_sec_IO < 5:
                            merge_insert_buffer() // merge at most 5 insert buffer
                        if buf_get_modified_ratio_pct > innodb_max_dirty_page_pct :
                            buffer_pool_flush_100_dirty_page()
                        if no user activity:
                            goto background_loop:
                    // do things once per 10 sec
                    if last_10_sec_IOs < 200:
                        buffer_pool_flush_100_dirty_page()
                    merge_insert_buffer() // merge at most 5 insert buffer
                    log_buffer_flush()    // flush log buffer to disk
                    full_purge()
                    if buf_get_modified_ratio_pct > 70%:
                        buffer_pool_flush_100_dirty_page()
                    else:
                        buffer_pool_flush_10_dirty_page()
                    goto loop
                    background_loop:
                        do things
                        goto loop
            2. 后台循环(background loop) *原书中background拼成了backgroup
                当没有用户活动(数据库空闲时), 数据库关闭时(shutdown), 就会切换到这个循环, background loop包括:
                    a. 删除无用Undo page(always)
                    b. merge 20 insert buffer(always)
                    c. 跳回主循环(always)
                    d. 不断刷新100个页直到符合条件(sometime, 跳到flush loop中完成)   
            3. 刷新循环(flush loop)
                若flush loop中没有事情可以做, InnoDB会切换到suspend_loop, 将Master Thread挂起, 等待事情的发生, 如果用户启用了InnoDB引擎, 却没有使用任何InnoDB存储引擎的表, 那么Master Thread总是处于挂起的状态
            4. 暂停循环(suspend loop)
            
            最后, 完整的Master Thread伪代码:
            def master_thread()
                loop:
                for in range(10):
                    thread_sleep(1)    // sleep 1 sec
                    log_buffer_flush() // flush log buffer to disk
                    if last_one_sec_IO < 5:
                        merge_insert_buffer(5) // merge at most 5 insert buffer
                    if buf_get_modified_ratio_pct > innodb_max_dirty_page_pct :
                        buffer_pool_flush_100_dirty_page()
                    if no user activity:
                        goto background_loop:
                // do things once per 10 sec
                if last_10_sec_IOs < 200:
                    buffer_pool_flush_100_dirty_page()
                merge_insert_buffer(5) // merge at most 5 insert buffer
                log_buffer_flush()    // flush log buffer to disk
                full_purge()
                if buf_get_modified_ratio_pct > 70%:
                    buffer_pool_flush_100_dirty_page()
                else:
                    buffer_pool_flush_10_dirty_page()
                goto loop
                background_loop:
                    full_purge()
                    merge_insert_buffer(20) // merge at most 20 insert buffer
                    if not idle:
                        goto loop: 
                    else:
                        goto flush loop:
                flush loop:
                    buffer_pool_flush_100_dirty_page()
                    if buf_get_modified_ratio_pct > innodb_max_dirty_page_pct :
                        goto flush loop:
                    goto suspend loop:
                suspend loop:
                    suspend_thread()
                    waiting_event(goto loop)
####    Master Thread的参数(InnoDB 1.0x后的参数更新)
            1. innodb_io_capacity
                上面的伪代码中可以看到, 因为硬编码(hard coding)的原因InnoDB最多一次只能写入100页的dirty page到硬盘, 合并20个insert buffer, 由于硬盘性能的提升, 这个hard coding的IO写入量可能会变成写入的瓶颈, 发生宕机需要恢复时, 由于还有很多数据还没刷新到硬盘, 会导致恢复的时间可能需要更久, 尤其时对于insert buffer来说
                从InnoDB 1.0x后, 提供了innodb_io_capacity, 用来配置硬盘IO的吞吐量, 默认值为200, 对于刷新到硬盘的页的数量, 会按照innodb_io_capacity的百分比来控制:
                    a. merge insert buffer的数量为innodb_wio_capacity的5%
                    b. 从缓冲区刷新脏页时, 刷新脏页的数量为innodb_io_capacity
            2. innodb_max_dirty_pages_pct的默认值
                并且在InnoDB 1.0x前 innodb_max_dirty_pages_pct 默认为90, 意味着dirty page占据缓冲池的90%才刷新100页dirty page到硬盘, 如果有很大的内存, 或者数据库服务器压力很大时, 写入dirty page的速度反而会降低, 并且数据库恢复的阶段核能需要更多的时间
                经过实验, 合适的值时75(default)-80, 这样可以保证dirty page刷新频率和IO负载
            3. innodb_adaptive_flushing(自适应的刷新)
                该值影响每秒刷新dirty page的数量. 原来的刷新规则: dirty page占缓冲池的比例小于innodb_max_dirty_pages_pct时, 不刷新dirty page, 大于innodb_max_dirty_pages_pct时, 刷新100个dirty page. innodb_adaptive_flushing参数引入, InnoDB通过一个名为buf_flush_get_desired_flush_rate的函数来判断需要刷新dirty page的合适数量, buf_flush_get_desired_flush_rate通过判断产生redo log的速度来决定合适的dirty page刷新率, 因此, dirty page占缓冲池的比例小于innodb_max_dirty_pages_pct也会杀心一定量的dirty page
            4. innodb_purge_batch_size
                之前版本的full purge操作时, 最多收回20个Undo page, 引入innodb_purge_batch_size后, 可以改变该值来控制每次full purge回收的Undo页的数量(default20, SET GLOBAL innodb_purge_batch_size=50可以动态设置)
####    观察Master Thread对Mysql负载判断
            SHOW ENGINE INNODB STATUS 中的Master Thread的状态信息
            srv_master_thread loops: [loop次数 1_second, 每秒过去次数 sleeps, 10秒操作次数 10_second, background loop次数 background, flush loop次数 flush]
            因为InnoDB的内部优化, 压力大时并不总等待1秒再进行1秒操作, 因此并不能认为 1_second 和 sleeps的值总是相等, 在默写情况下, 可以通过两者的差值来判断数据库的负载压力
####    InnoDB 1.2X 版本后的Master Thread
            InnoDB 1.2x后对Master Thread再次进行了优化
            if InnoDB is idle:
                srv_master_do_idle_tasks()
            else
                srv_master_do_active_tasks()
            其中srv_master_do_idle_tasks()就是之前版本的10秒操作, srv_master_do_active_tasks()就是之前的1秒操作, 并且, 对于刷新dirty page的操作从Master Thread中分离到了单独的Page Cleaner Thread中, 减轻了Master Thread的工作
