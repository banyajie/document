#### mysql 常见面试题

##### 事务的四大特性（ACID）？

- 原子性
- 一致性
- 隔离性
- 持久性

##### 什么是死锁？如何解决？

- 死锁
  - xx
- 解决方案
  - 如果不同程序会并发存取多个表， 尽量约定以相同的顺序访问表，可以大大降低死锁机会。
  - 在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁产生概率
  - 对于非常容易产生死锁的业务部分，可以尝试使用升级锁定颗粒度，通过表级锁定来减少死锁产生的概率；
  - 如果业务处理不好可以用分布式事务锁或者使用乐观锁



##### 什么是脏读、幻读、不可重复读？

- 脏读：某个事务已更新一份数据，另一个事务在此时读取了同一份数据， 由于某些原因，前一个RollBack了操作，则后一个事务所读取的数据就会是不正确的
- 幻读：在一个事务的两次查询中数据行数不一致， 例如有一个事务查询了几列(Row)数据， 而另一个事务却在此时插入了新的几列数据，先前的事务在接下来的查询中， 就会发现有几列数据是它先前所没有的。
- 不可重复读：在一个事务的两次查询之中数据不一致， 这可能是两次查询过程中间插入了一个事务更新的原有的数据。



##### Mysql数据库 cpu 到100%的话怎么处理？

- show processlist
- 观察是否有消耗比较高的session、慢查询、执行计划是否准确、index是否缺失



##### b树和b+树有什么区别，为什么Mysql使用B+树？

- b树
  - 节点排序
  - 一个节点下面可以有多个数据、多个数据也是排好序的
  - 
- b+树
  - 拥有b树的特点
  - 叶子节点包含所有的数据节点
  - 叶子节点有关系（有指针联系起来）。可以只通过叶子节点找到所有数据





- Innodb是如何支持范围查找可以走索引？

  > select * from where a > 6;
  >
  > 1：select * from where  a = 6
  >
  > 2：后面全部数据返回

- 最左前缀原则是什么？

- 为什么要遵从最左前缀原则才可以利用到索引？

- 范围查找导致索引失效的原理？

- 覆盖索引的底层原理？

- 索引扫描原理？

- order by 是如何工作的？

  - 全自动排序
  - rowid排序
  - 

- order by 为什么可以引起索引失效？

- Mysql 日志（redo log、undo log、binlog、）



##### Mysql 锁有哪些，如何理解？

> 按照锁粒度分
>
> - 行锁 - 锁一行数据，粒度小。并发度高
> - 表锁 - 锁整张表，粒度大，并发度低
> - 间隙锁 - 锁一个区间
>
> 还可以分为
>
> - 共享锁 - 读锁 - 一个事务给某行数据加了读锁。不影响其他事务的读取。但是不能写数据
> - 排他锁 - 写锁
>
> 还可以分为：
>
> - 乐观锁 - 并不会真正的锁数据，通过版本号实现
> - 悲观锁 - 上面所有的行锁、表锁都是悲观锁



##### Mysql 是如何保证数据不丢的？

> WAL（Write-Ahead Logging） 机制。预写日志系统
>
> 只要redo log 和 binlog 能保证持久化磁盘。就可以确保Mysql 异常重启后，数据可以恢复。
>
> 
>
> So：如何保证redo log 和 binlog 一定持久化磁盘
>
> 
>
> redolog 写入机制？
>
> - redolog buffer。事务执行过程中，生成的redolog要先写入到redolog buffer中
> - 
>
> binlog 写入机制？
>
> - binLog的写入比较简单，事务执行过程中，先把日志写入binlog cache，事务提交的时候，再把binlog cache 写入binlog文件中
> - cache 写入 biglog文件的时机
>   - sync_binlog = 0 ，每次提交事务都只写cache。不sync binglog 文件
>   - sync_binlog = 1，每次提交都会刷新cache到binlog文件
>   - sync_binlog = N （N > 1）。每次提交都写入cache。等累计到N个事务后刷新到binlog。
> - 因此：在出现IO瓶颈的场景里。可以通过提高sync_binglog提升性能，但是如果主机发生异常重启。会丢失N个事务的信息



##### Mysql 是如何保证主备一致性的？

> 主备同步的基本原理？
>
> ![image-20220523220538737](/Users/banyajie/Library/Application Support/typora-user-images/image-20220523220538737.png)
>
> - master节点线程 dump_thread 读取binlog发送给 slave 节点
>
> - slave节点维护两个线程：io_thread 和 sql_thread 
>
>   - io_thread 负责和主库建立连接，接收master数据
>   - io_thread 接收到数据后，写到本地文件 relay log（中转日志）
>   - sql_thread 读取relay log日志，解析命令并执行
>
>   
>
> BinLog 的格式？
>
> - Binlog 里面到底是什么内容，为什么从库可以拿过来直接执行？
>
> 三种格式
>
> ```mysql
> #  执行语句
> deletefromt/*comment*/ wherea>=4andt_modified<='2018-11-10'limit1
> 
> # 查看binlog
> show binlog events in 'master.000001';
> ```
>
> 
>
> - statement
>
>   - ```mysql
>     # 格式
>     SET @@SESSION.GTID_NEXT='ANONYMOUS’
>     BEGIN
>     user `test`, delete from t where a >= 4 limit 1;    # 执行的sql语句
>     commit
>     ```
>
>   - statement 记录的是语句执行的原语。会导致主备不一致的情况，原因：在主库和备库执行同一条语句的时候用到的索引不一定相同。
>
>   - 
>
> - row
>
>   - ```mysql
>     SET @@SESSION.GTID_NEXT='ANONYMOUS’
>     BEGIN
>     table_id: 226 (test.t)
>     table_id: 225 flags: STMT_END_F
>     commit
>     ```
>
>   - row格式的binlog里没有了SQL语句的原文，而是替换成了两个event:Table_map和Delete_rows。
>
>     - Table_mapevent：用于说明接下来要操作的表是test库的表t
>     - Delete_rowsevent，用于定义删除的行为
>
>   - 当binlog_format使用row格式的时候，binlog里面记录了真实删除行的主键id，这样 binlog传到备库去的时候，就肯定会删除id=4的行，不会有主备删除不同行的问题。
>
>   
>
> - mixed（前两种的混合）
>
>   - 为什么会有mixed格式的binlog呢？
>     - 因为有些statement格式的binlog可能会导致主备不一致，所以要使用row格式
>     - 但row格式的缺点是，很占空间。比如你用一个delete语句删掉10万行数据，用statement的 话就是一个SQL语句被记录到binlog中，占用几十个字节的空间。但如果用row格式的binlog， 就要把这10万条记录都写到binlog中。这样做，不仅会占用更大的空间，同时写binlog也要耗 费IO资源，影响执行速度。
>     - 所以，MySQL就取了个折中方案，也就是有了mixed格式的binlog。mixed格式的意思 是，MySQL自己会判断这条SQL语句是否可能引起主备不一致，如果有可能，就用row格式， 否则就用statement格式。



##### Mysql 是如何保证高可用的？

> - 主备延迟问题？
>
>   - 
>   - 主备延迟最直接的表现是，备库消费中转日志（relay log）的速度比主库产生binlog的速度慢
>     - 首先，有些部署条件下，备库所在机器的性能要比主库所在的机器性能差
>     - 第二种常见的可能了，即备库的压力大，如果不注重备库的性能。在备库上做数据分析占用cpu
>       - 一主多从
>       - 通过binlog输出到外部系统，比如Hadoop这类系统，让外部系统提供统计类查询的能力
>     - 大事务会导致主备延迟
>       - 比如一个事务在master节点执行10分钟，同步到slave节点也需要执行10分钟。那主备不一致就会持续10分钟
>     - 
>
> - 由于主备延迟的存在，所以主备切换的时候有不同的策略
>
>   - 可靠性优先策略下主备切换过程（存在不可用时间）
>
>     - 1：判断备库B现在的延迟时间 seconds_behind_master ，如果小于（<5s）某个值就继续下一步，否则就继续重试这一步
>     - 2：把主库 A 设置成 readonly 只读状态
>     - 3：判断 B 库的 seconds_behind_master 的值，直到这个值为0
>     - 4：把B库改为读写状态
>     - 5：切换业务请求到B库
>
>     
>
>   - 可用性优先策略（尽量降低不可用时间），可能会导致数据不一致情况
>
>     - 1：把B库改为读写状态
>     - 2：切换业务请求到B库
>     - 3：同上
>
>     

##### Mysql 慢查询如何优化？

> - 检查是否走了索引
> - 检查执行语句利用的索引是否是最优索引
> - 检查执行语句查询字段是否是必须的，是否查询了过多字段，查出了多余数据
> - 检查表内容是否过多，是否需要分库分表
> - 检查db 实例机器性能配置



##### Mysql 数据库中，什么情况下设置了索引但是用不到索引？

> - 没有符合最左前缀原则
>   - 最左前缀原则？
>     - 
> - 字段进行了隐式类型转换
> - 走索引没有走全表查询速度快
> - 运算条件



##### MyISAM 和 Innodb的区别是什么？

> - Innodb支持事务，MyIsam不支持
> - Innodb支持外键，MyISam不支持
> - Innodb是聚簇索引，MyISam是非聚簇索引
>   - 聚簇索引的文件存放在主键索引的叶子节点上，因此Innodb必须有主键，通过主键索引效率高。
>   - MyIsam是非聚簇索引，数据文件是分离的，索引保存的是数据的指针
> - Innodb不保存具体的行数据，执行 select count(*) from table 式需要全表扫描。而MyIsam用一个变量保存。
> - Innode的锁粒度有行锁。MyIsam最小粒度是表锁
> - 



