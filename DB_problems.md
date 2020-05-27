### Database 问题

锁
    行锁
        update row => where 条件中锁index命中的rows
                      如果没有命中index, 会锁表

索引
    全文索引
        全文索引和like

离散读(db file scattered read)

##### 并发问题   
    