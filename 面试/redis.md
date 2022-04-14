####  Redis



##### 数据结构和实现

###### string

>   SDS结构
>
>   优化长度读写O(1)，杜绝缓冲区溢出，减少修改字符串时内存重新分配次数（空间预分配、惰性回收）
>
>   ```c
>   // 动态字符串
>   struct sdshdr{
>        //记录buf数组中已使用字节的数量
>        //等于 SDS 保存字符串的长度
>        int len;
>        //记录 buf 数组中未使用字节的数量
>        int free;
>        //字节数组，用于保存字符串
>        char buf[];
>   }
>   
>   ```
>
>   

###### list

>   压缩列表(zip-list)、双向链表(linked list)
>
>   创建list的时候默认使用redis_encoding_ziplist编码， 但是有时候压缩列表会转换为双向链表
>
>   -   向列表中加入一个字符串值，而且这个字符串值的长度超过server.list_max_ziplist_value （默认 64 ）
>   -   ziplist 节点数量超过 server.list_max_ziplist_entries （默认 512）
>
>   ```c
>   // 3.2版本之前 - 压缩列表（使用连续内存，每个节点（entry）都是连续存储的）
>   // 优点：节约内存
>   // 缺点：数据量大的时候存取效率低，只能顺序读取
>   
>   // ziplis 存储结构 - 一段连续的内存存储
>   area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|
>   
>   size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
>               +---------+--------+-------+--------+--------+--------+--------+-------+
>   component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
>               +---------+--------+-------+--------+--------+--------+--------+-------+
>                                          ^                          ^        ^
>   address                                |                          |        |
>                                   ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
>                                                                     |
>                                                            ZIPLIST_ENTRY_TAIL
>   
>   // zlbytes：整个ziplist占用的内存字节数，对ziplist进行内存重新分配或者计算末端时使用
>   // zltail：到达ziplist尾节点的偏移量。可以不遍历ziplist，弹出尾部节点
>   // zllen：节点数量
>   ..
>   // zlend：标记
>       
>       
>   // 节点 - 变长编码
>   typedef struct zlentry {   
>       unsigned int prevrawlensize, prevrawlen;  // prevrawlen是前一个节点的长度，prevrawlensize是指prevrawlen的大小，有1字节和5字节两种
>       unsigned int lensize, len;  			  // len为当前节点长度 lensize为编码len所需的字节大小
>       unsigned int headersize;    			  // 当前节点的header大小
>       unsigned char encoding;  				  // 节点的编码方式
>       unsigned char *p;   					  // 指向节点的指针
>   } zlentry;
>   
>   void zipEntry(unsigned char *p, zlentry *e) {   				// 根据节点指针返回一个enrty
>       ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);    // 获取prevlen的值和长度
>       ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);  // 获取当前节点的编码方式、长度等
>       e->headersize = e->prevrawlensize + e->lensize; 			// 头大小
>       e->p = p;
>   }
>   
>   // 问题
>   // 
>   // 如何通过一个节点向前跳转到另外一个节点 ？
>   // 用指向当前节点的指针 e ， 减去 前一个 entry的长度， 得出的结果就是指向前一个节点的地址 p 
>   
>   // ziplist 连锁更新问题
>   // 
>   ```
>
>   ```c
>   // 3.2版本之后 -  quicklist(ziplist和linked list 的结合)
>   // 由ziplist组成的双向链表
>   typedef struct quicklistNode {
>       struct quicklistNode *prev; //上一个node节点
>       struct quicklistNode *next; //下一个node
>       unsigned char *zl;           // 保存的数据 压缩前ziplist 压缩后压缩的数据
>       unsigned int sz;             /* ziplist size in bytes */
>       unsigned int count : 16;     /* count of items in ziplist */
>       unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
>       unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
>       unsigned int recompress : 1; /* was this node previous compressed? */
>       unsigned int attempted_compress : 1; /* node can't compress; too small */
>       unsigned int extra : 10; /* more bits to steal for future usage */
>   } quicklistNode;
>   
>   typedef struct quicklistLZF {
>       unsigned int sz; /* LZF size in bytes*/
>       char compressed[];
>   } quicklistLZF;
>   
>   typedef struct quicklist {
>       quicklistNode *head; //头结点
>       quicklistNode *tail; //尾节点
>       unsigned long count;        /* total count of all entries in all ziplists */
>       unsigned int len;           /* number of quicklistNodes */
>       int fill : 16;              /* fill factor for individual nodes *///负数代表级别，正数代表个数
>       unsigned int compress : 16; /* depth of end nodes not to compress;0=off *///压缩级别
>   } quicklist;
>   
>   
>   ```
>
>   ```c
>   // linked list（双向链表）
>   
>   typedef  struct listNode{
>       //前置节点
>       struct listNode *prev;
>       //后置节点
>       struct listNode *next;
>       //节点的值
>       void *value;  
>   } listNode
>       
>   typedef struct list{
>           //表头节点
>           listNode *head;
>           //表尾节点
>           listNode *tail;
>           //链表所包含的节点数量
>           unsigned long len;
>           //节点值复制函数
>           void (*free) (void *ptr);
>           //节点值释放函数
>           void (*free) (void *ptr);
>           //节点值对比函数
>           int (*match) (void *ptr,void *key);
>   }list;
>   ```
>
>   问题
>
>   -   list是如何实现阻塞队列（消费者读取数据时，如果队列为空，则阻塞）（blpop、brpop、brpoplpush）？
>
>       

###### hash

>   redis的hash架构就是标准的hashtable的结构，通过挂链解决冲突问题。
>
>   ![img](https://img-blog.csdnimg.cn/20190623214252310.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21jY2FuZDEyMzQ=,size_16,color_FFFFFF,t_70)
>
>   ```c
>   // hash表结构定义：
>   
>   typedef struct dict {
>       dictType *type;		// 数据类型，根据不同的类型对应不同的回调函数
>       void *privdata;
>       dictht ht[2];   /* 两个hash表,rehash时使用*/
>       long rehashidx; /* rehash的索引, -1表示没有进行rehash */
>       int iterators;  /*  */
>   } dict;
>   
>   typedef struct dictht{
>        //哈希表数组
>        dictEntry **table;   ==  》 dictEntry
>        //哈希表大小
>        unsigned long size;
>        //哈希表大小掩码，用于计算索引值
>        //总是等于 size-1
>        unsigned long sizemask;
>        //该哈希表已有节点的数量
>        unsigned long used;
>   }
>   
>   // hash表由数组table组成，table中每个元素都是指向 dictEntry结构
>   typedef struct dictEntry{
>        //键
>        void *key;
>        //值
>        union{
>             void *val;
>             uint64_tu64;
>             int64_ts64;
>        }v;
>    
>        //指向下一个哈希表节点，形成链表
>        struct dictEntry *next;
>   }
>       
>   
>   ```
>
>   -   rehash过程、扩容、缩容
>       -   

###### set

>   根据数据类型不同底层存储结构不同
>
>   -   数组，存储时是有序的，使用二分查找
>       -   所有数据都是整数类型
>       -   数量量不超过512个
>   -   hashtable
>       -   
>
>   ```c
>   // 整数类型的
>   typedef struct intset {
>       // 编码方式
>       uint32_t encoding;
>       // 集合包含的元素数量
>       uint32_t length;
>       // 保存元素的数组
>       int8_t contents[];
>   } intset;
>   
>   
>   ```
>
>   

###### zset

>   根据数据量的不同选择：压缩列表、跳跃表
>
>   跳跃表结构图
>
>   ![img](https://upload-images.jianshu.io/upload_images/10204326-40ef4fe6ffb397a3.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/775/format/webp)
>
>   -   层数的选择算法
>
>   -   跳跃表的高度
>       -   n个元素的跳跃表，每个元素插入的时候都要做一次试验，用来决定元素占据的层数 K ，跳跃表的高度等于其中产生的最大 K
>
>   
>
>   -   为什么使用跳跃表而不是使用平衡树？
>       -   
>
>   
>
>   ```c
>   // 跳跃表是一种有序的数据结构，通过每个节点维护多个指向其他节点的指针，从而达到快速访问的目的。
>   
>   // 跳跃表节点
>   typedef struct zskiplistNode {
>       //层
>       struct zskiplistLevel{
>           //前进指针
>           struct zskiplistNode *forward;
>           //跨度
>           unsigned int span;
>       }level[];
>   
>       //后退指针
>       struct zskiplistNode *backward;
>       //分值
>       double score;
>       //成员对象
>       robj *obj;
>   } zskiplistNode
>       
>   // 跳跃表    
>   typedef struct zskiplist{
>           //表头节点和表尾节点
>           structz skiplistNode *header, *tail;
>           //表中节点的数量
>           unsigned long length;
>           //表中层数最大的节点的层数
>           int level; 
>   }zskiplist;
>   
>   /**
>    * 有序集合结构体
>    */
>   typedef struct zset {
>       /*
>        * Redis 会将跳跃表中所有的元素和分值组成 
>        * key-value 的形式保存在字典中
>        * todo：注意：该字典并不是 Redis DB 中的字典，只属于有序集合
>        */
>       dict *dict;
>       /*
>        * 底层指向的跳跃表的指针
>        */
>       zskiplist *zsl;
>   } zset;
>   
>   
>   
>   // 数据操作过程
>   // 1：搜索过程
>   // 2：插入过程
>   // a：随机选择当前节点占据的层数 k 
>   // b：然后在 L1- Lk层都插入当前数据，如果 k 大于当前的最大层，增加一层
>   // 3：删除过程
>   ```
>
>   



##### Redis 过期策略



###### 内存淘汰策略

>   说明：内存淘汰策略为 Redis 在内存不足时，怎么处理需要写入并且需要申请内存的请求
>
>   
>
>   -   noeviction：内存不足时，写入报错
>   -   allkeys-lru：内存不足时，写入数据时，在键空间中，移除最少使用的key
>   -   allkeys-random：内存不足时，写入数据时，在键空间中，随机移除某个key
>   -   valatile-lru：当内存不足时，写入数据时，在设置了过期时间的键空间中，移除最少使用的key
>   -   volatile-random：当内存不足时，写入数据时，在设置了过期时间的键空间中，随机移除某个key
>   -   volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除
>   -   



###### 过期策略（过期数据删除的策略）

>   -   定时删除
>       -   在为key设置过期时间的同时，为当前 key 启动一个定时器，定时器定时生效时，对 key 进行删除
>       -   优点
>           -   保证内存被及时释放
>       -   缺点：
>           -   影响性能（需要为每个需要过期的key维护一个定时器）
>           -   如果过期key太多，删除过期key造成cpu消耗
>
>   -   惰性删除
>       -   key过期的时候不删除，每次从数据库获取key的时候去检查是否过期，若过期，则删除，返回null
>       -   优点：
>           -   删除操作只发生在读取key时，只删除当前key对cpu影响较小
>       -   缺点
>           -   如果大量key过期后不会再被读取，会造成内存浪费
>   -   定期删除
>       -   每隔一段时间执行一次删除(在redis.conf配置文件设置hz，1s刷新的频率)过期key操作
>       -   优点
>           -   通过限制删除操作的时长和频率，来减少删除操作对CPU时间的占用--处理"定时删除"的缺点
>           -   定期删除过期key--处理"惰性删除"的缺点
>       -   缺点：
>           -   在内存友好方面，不如"定时删除"
>           -   在CPU时间友好方面，不如"惰性删除"
>       -   难点
>           -   合理设置删除操作的执行时长
>
>   
>
>   Redis 目前选择是定期删除 + 惰性删除



###### RDB文件对过期key的处理

>   过期key对RDB文件无影响
>
>   -   从内存数据库持久化到RDB文件时，写入RDB时，会判断key是否过期
>   -   从RDB恢复到内存数据库时，会检查key是否过期，过期不处理



###### AOF文件对过期key的处理

>   过期key对AOF无影响
>
>   -   从内存数据库持久化数据到AOF文件
>       -   当key过期后，还没有被删除，此时进行持久化操作key不会写入AOF文件
>       -   当key过期后，已经删除。删除的时候会向AOF文件总追加一个 del key 的操作



##### Redis 持久化 与 事务

官方地址：https://redis.io/topics/persistence



###### RDB

>   快照机制（以二进制方式保存到磁盘）
>
>   
>
>   原理：把当前内存中的数据集快照写入磁盘，也就是snapshot快照（数据库中所有的键值对数据）。恢复时将快照文件直接读取到内存中
>
>   触发方式：
>
>   -   自动触发
>
>       -   save m n：表示m秒内数据集存在n次修改时，自动触发bgsave。如果redis是缓存功能的话，可以不持久化
>       -   **stop-writes-on-bgsave-error**：保存数据失败后是否还继续接收请求
>       -   **rdbcompression**”：压缩
>       -   **rdbchecksum**：数据校验
>       -   **dbfilename**：文件名
>       -   dir：文件目录
>
>   -   手动触发
>
>       -   save：该命令会阻塞当前redis服务器，执行save命令期间，redis不可以处理其他命令，知道RDB过程完成
>       -   bgsave：后台执行RDB save。不阻塞redis服务器（fork期间阻塞，阻塞时间很短）
>
>       

>   -   优点
>
>       -   只有一个RDB文件，方便持久化
>
>       -   性能最大化，独立子进程处理，不影响主进程数据处理
>
>       -   数据量大时，比AOF效率高
>
>           
>
>   -   缺点
>
>       -   因为是定时同步，所以有丢失数据的可能
>
>       

###### AOF（append only file）

>   原理：以协议文本的方式，将所有数据库进行写入的命令（及其参数）记录到AOF文件中，以此达到记录数据库的状态
>
>   过程
>
>   -   命令传播：Redis 将执行完的命令、命令的参数、命令的参数个数等信息发送到 AOF 程序中
>       -   
>   -   缓存追加：AOF 程序根据接收到的命令数据，将命令转换为网络通讯协议的格式，然后将协议内容追加到服务器的 AOF 缓存中
>       -   AOF程序接受命令、命令的参数、以及参数的个数、所使用的数据库等信息
>       -   将命令还原成 Redis 网络通讯协议
>       -   将协议文本追加到 `aof_buf` 末尾
>   -   文件写入和保存：AOF 缓存中的内容被写入到 AOF 文件末尾，如果设定的 AOF 保存条件被满足的话， `fsync` 函数或者 `fdatasync` 函数会被调用，将写入的内容真正地保存到磁盘中
>       -   WRITE：根据条件，将 `aof_buf` 中的缓存写入到 AOF 文件。
>       -   SAVE：根据条件，调用 `fsync` 或 `fdatasync` 函数，将 AOF 文件保存到磁盘中。
>
>   AOF保存模式：
>
>   -   AOF_FSYNC_NO：不保存
>   -   AOF_FSYNC_EVERYSEC：每一秒钟执行一次
>   -   AOF_FSYNC_ALWAYS：每执行一个命令执行一次
>
>   AOF重写
>
>   -   创建一个新的 AOF 文件来代替原有的 AOF 文件， 新 AOF 文件和原有 AOF 文件保存的数据库状态完全一样， 但新 AOF 文件的体积小于等于原有 AOF 文件的体积
>   -   

###### 事务

>   原子性：redis的事务是非原子性的、不支持回滚
>
>   过程：    
>
>   -   MULTI：开启事务，redis将后续的命令逐个放入队列中，然后使用 EXEC 命令原子化执行   
>   -    EXEC：执行事务中的所有操作指令    
>   -   DISCARD：取消事务    
>   -   WATCH：监视一个或者多个key，如果这个事务在执行前。key被修改，则事务中断。不会执行任何命令    
>   -   UNWATCH：取消对key的监视
>
>   事务过程中失败处理：
>
>   -   语法错误（编译器错误：比如命令拼写错误）
>       -   如：同时修改key1、key2        
>       -   multi        
>       -   set key1 v1        sett key2 v2 （在此阶段就中止事务）                
>       -   保留原值（都未修改成功），事务中止
>   -   redis类型错误（运行时错误
>       -   如：        multi        set key1 v1        lpush key2 v2（key2不是list）        exec        
>       -   执行时出错，事务不中止。跳过出错的那条。继续执行
>
>   

##### Redis 集群方案

###### 主从复制模式

>   ![redis-master-slave](https://img2020.cnblogs.com/other/632381/202003/632381-20200316092434553-1122086987.png)
>
>   主从复制模式中包含了一个 master 实例和一个或者多个 Slave 实例。客户端对 master 节点进行读写操作，对 slave 节点进行读操作。master 节点写入的数据会实时自动同步给 slave 节点
>
>   工作机制：
>
>   -   slave启动后，向master发送SYNC命令，master接收到SYNC命令后通过bgsave保存快照（即上文所介绍的RDB持久化），并使用缓冲区记录保存快照这段时间内执行的写命令
>   -   master将保存的快照文件发送给slave，并继续记录执行的写命令
>   -   slave接收到快照文件后，加载快照文件，载入数据
>   -   master快照发送完后开始向slave发送缓冲区的写命令，slave接收命令并执行，完成复制初始化
>   -   此后master每次执行一个写命令都会同步发送给slave，保持master与slave之间数据的一致性
>
>   优点；
>
>   -   可以读写分离，分担 master 节点的压力
>   -   master、slave节点之间的同步是非阻塞的。同步期间，依然可以接收处理客户端的请求
>
>   缺点
>
>   -   不具备自动容错与恢复功能，master或slave的宕机都可能导致客户端请求失败，需要等待机器重启或手动切换客户端IP才能恢复
>   -   master宕机，如果宕机前数据没有同步完，则切换IP后会存在数据不一致的问题
>   -   难以支持在线扩容，Redis的容量受限于单机配置

###### Sentinel（哨兵）模式

>   基于主从复制模式，引入了哨兵机制以及自动处理故障
>
>   ![redis-sentinel](https://img2020.cnblogs.com/other/632381/202003/632381-20200316092434904-227928571.png)
>
>   哨兵的作用：
>
>   -   监控master、slave是否正常运行
>   -   当master出现故障时，能自动将一个slave转换为master
>   -   多个哨兵可以监控同一个Redis，哨兵之间也会自动监控
>
>   工作机制
>
>   -   sentinel monitor <master-name> <ip> <redis-port> <quorum>：定位master的IP、端口，一个哨兵可以监控多个master数据库
>   -   。。。
>
>   优点
>
>   -   哨兵模式基于主从复制模式，所以主从复制模式有的优点，哨兵模式也有
>   -   哨兵模式下，master挂掉可以自动进行切换，系统可用性更高
>
>   缺点
>
>   -   难以在线扩容的，Redis的容量受限于单机配置
>   -   需要额外的资源来启动sentinel进程，实现相对复杂一点，同时slave节点作为备份节点不提供服务
>   -   

###### Redis Cluster模式

>   哨兵模式解决了主从复制不能自动故障转移，达不到高可用的问题，但还是存在难以在线扩容，Redis容量受限于单机配置的问题。Cluster模式实现了Redis的分布式存储，即每台节点存储不同的内容，来解决在线扩容的问题
>
>   ![redis-cluster](https://img2020.cnblogs.com/other/632381/202003/632381-20200316092435392-508884462.png)
>
>   Cluster采用无中心结构，特点：
>
>   -   所有的redis节点彼此互联(PING-PONG机制)，内部使用二进制协议优化传输速度和带宽
>   -   节点的fail是通过集群中超过半数的节点检测失效时才生效
>   -   客户端与redis节点直连,不需要中间代理层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可
>
>   工作机制
>
>   -   在Redis的每个节点上，都有一个插槽（slot），取值范围为0-16383
>   -   当我们存取key的时候，Redis会根据 CRC16 的算法得出一个结果，然后把结果对 16384 求余数，这样每个key都会对应一个编号在 0-16383 之间的哈希槽，通过这个值，去找到对应的插槽所对应的节点，然后直接自动跳转到这个对应的节点上进行存取操作
>   -   为了保证高可用，Cluster模式也引入主从复制模式，一个主节点对应一个或者多个从节点，当主节点宕机的时候，就会启用从节点
>   -   当其它主节点ping一个主节点A时，如果半数以上的主节点与A通信超时，那么认为主节点A宕机了。如果主节点A和它的从节点都宕机了，那么该集群就无法再提供服务了
>
>   
>
>   Cluster模式集群节点最小配置6个节点(3主3从，因为需要半数以上)，其中主节点提供读写操作，从节点作为备用节点，不提供请求，只作为故障转移使用
>
>   
>
>   部署
>
>   ```shell
>   port 7100 		# 本示例6个节点端口分别为7100,7200,7300,7400,7500,7600 
>   daemonize yes 	# r后台运行 
>   pidfile /var/run/redis_7100.pid 	# pidfile文件对应7100,7200,7300,7400,7500,7600 
>   cluster-enabled yes 				# 开启集群模式 
>   masterauth passw0rd 				# 如果设置了密码，需要指定master密码
>   cluster-config-file nodes_7100.conf # 集群的配置文件，同样对应7100,7200等六个节点
>   cluster-node-timeout 15000 			# 请求超时 默认15秒，可自行设置 
>   
>   # 启动实例
>   redis-server redis_7100.conf
>   
>   # 构造集群
>   redis-cli --cluster create --cluster-replicas 1 127.0.0.1:7100 127.0.0.1:7200 127.0.0.1:7300 127.0.0.1:7400 127.0.0.1:7500 127.0.0.1:7600 -a passw0rd
>   
>   
>   ```
>
>   优点
>
>   -   无中心架构，数据按照slot分布在多个节点
>   -   集群中的每个节点都是平等的关系，每个节点都保存各自的数据和整个集群的状态。每个节点都和其他所有节点连接，而且这些连接保持活跃，这样就保证了我们只需要连接集群中的任意一个节点，就可以获取到其他节点的数据。
>   -   可线性扩展到1000多个节点，节点可动态添加或删除
>   -   能够实现自动故障转移，节点之间通过gossip协议交换状态信息，用投票机制完成slave到master的角色转换
>
>   缺点
>
>   -   客户端实现复杂，驱动要求实现Smart Client，缓存slots mapping信息并及时更新，提高了开发难度。目前仅JedisCluster相对成熟，异常处理还不完善，比如常见的“max redirect exception”
>   -   节点会因为某些原因发生阻塞（阻塞时间大于 cluster-node-timeout）被判断下线，这种failover是没有必要的
>   -   数据通过异步复制，不保证数据的强一致性
>   -   slave充当“冷备”，不能缓解读压力
>   -   批量操作限制，目前只支持具有相同slot值的key执行批量操作，对mset、mget、sunion等操作支持不友好
>   -   key事务操作支持有线，只支持多key在同一节点的事务操作，多key分布不同节点时无法使用事务功能
>   -   不支持多数据库空间，单机redis可以支持16个db，集群模式下只能使用一个，即db 0

###### Codis

>   分布式redis解决方案
>
>   codis是一个代理中间件
>
>   <img src="https://images.xiaozhuanlan.com/photo/2019/d6a5ba5f74b9eeef7a6dd3784e3627fe.png" alt="img" style="zoom:33%;" />
>
>   Codis底层会处理请求的转发、不停机的数据迁移等工作，对于前面的客户端来说，Codis是透明的，可以简单地认为客户端（client）连接的是一个内存无限大的Redis服务。
>
>   
>
>   codis架构图
>
>   <img src="https://images.xiaozhuanlan.com/photo/2019/5f542e942fc35b9a042476ab917a193b.png" alt="img" style="zoom:67%;" />
>
>   四个组成部分：
>
>   -   Codis proxy（实现redis协议，无状态）
>       -   工作原理
>           -   
>   -   Codis Dashboard（codis config）
>       -   集群的管理工具，所有对集群的操作包括proxy、server的添加、删除、数据迁移都必须通过dashboard完成。Dashboard的启动过程是对一些必要的数据结构以及集群的操作的初始化
>       -   启动过程
>           -   New()阶段
>               -   启动时，首先读取配置文件，填充config信息。coordinator的值如果是"zookeeper"或者是"etcd"，则创建一个zk或者etcd的客户端。根据config创建一个Topom{}对象。Topom{}十分重要，该对象里面存储了集群中某一时刻所有的节点信息(slot，group，server等)，而New()方法会给Topom{}对象赋值。
>               -   随后启动18080端口，监听、处理对应的api请求。
>               -   最后启动一个后台线程，每隔一分钟清理pool中无效client。
>           -   Start()阶段
>               -   Start()阶段，将内存中model.Topom{}写入zk，路径是/codis3/codis-demo/topom
>               -   设置topom.online=true。
>               -   随后通过Topom.store从zk中重新获取最新的slotMapping、group、proxy等数据填充到topom.cache中（topom.cache，这个缓存结构，如果为空就通过store从zk中取出slotMapping、proxy、group等信息并填充cache。不是只有第一次启动的时候cache会为空，如果集群中的元素（server、slot等等）发生变化，都会调用dirtyCache，将cache中的信息置为nil，这样下一次就会通过Topom.store从zk中重新获取最新的数据填充。）
>               -   最后启动4个goroutine for循环来处理相应的动作 。
>           -   创建Group
>           -   创建codis server
>           -   
>   -   Codes Redis（codis-server）
>       -   codis维护的redis分支，加入了slot的支持和原子的数据迁移指令。
>   -   Zookeeper、etcd
>       -   集群元数据的存储（slot信息）
>
>   优点：
>
>   -   对客户端透明,与codis交互方式和redis本身交互一样
>   -   支持在线数据迁移,迁移过程对客户端透明有简单的管理和监控界面
>   -   支持高可用,无论是redis数据存储还是代理节点
>   -   自动进行数据的均衡分配
>   -   最大支持1024个redis实例,存储容量海量
>   -   高性能
>
>   
>
>   codis的不足：
>
>   -   欠缺安全考虑，codis fe页面没有登录验证功能
>   -   缺乏自带的多租户方案
>   -   缺乏集群缩容方案
>
>   改进
>
>   -   采用squid代理的方式来简单限制fe页面的访问，后期基于fe进行二次开发来控制登录
>   -   小业务通过在key前缀增加业务标识，复用相同集群；大业务使用独立集群，独立机器
>   -   采用手动迁移数据、腾空节点、下线节点的方法来缩容
>   -   



###### Twemproxy

>   proxy模式，无法平滑扩容



##### 常见面试题

###### 分布式锁

>   -   常见方案和问题
>
>   ```shell
>   
>       大部分实现方案：setnx + lua 或者 set key value（唯一值） px millsecond nx
>       
>       例如：（在单节点上实现分布式锁）
>           加锁：
>               set resource_name unique_name NX PX 3000     
>           
>           释放锁；(lua 脚本中，一定要比较value，防止误解锁)
>               if redis.call("get", KEYS[1]) == ARGV[1] then 
>                   return redis.call("del", KEYS[1])
>               else
>                  return 0
>      
>      三大要点：
>          命令要用：set key value  px millsecond nx
>          value：value要唯一
>          释放锁时要检查value：避免误解锁
>         
>   缺点：
>       加锁时只能加到一个redis节点上，即使redis通过sentinel保证高可用，如果master节点发生主从切换。会发生锁丢失的情况
>       例如：
>           1：在Redis的master节点上拿到了锁；
>           2：但是这个加锁的key还没有同步到slave节点；
>           3：master故障，发生故障转移，slave节点升级为master节点；
>           4：导致锁丢失。   
>                                      
>   合理的过期时间：
>       1：锁的过期时间小于业务处理时间该如何续期
>           watchdog 自动续期：设置锁时开启检查线程，过期前又没有释放的情况下重置过期时间
>           
>   ```
>
>   -   redLock（红锁）
>
>   ```shell
>   Redis的分布式环境中，我们假设有N个Redis master。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。我们确保将在N个实例上使用与在Redis单实例下相同方法获取和释放锁。现在我们假设有5个Redis master节点，
>   同时我们需要在5台服务器上面运行这些Redis实例，这样保证他们不会同时都宕掉。
>   
>   安全特性；永远只有一个client拿到锁
>   避免死锁：最终client都可能拿到锁，不会出现死锁的情况，即使原本锁住某资源的 client crash 或者 出现了网络分区
>   容错性：只要大部分redis节点存活就可以保证可用性
>   
>   客户端过程：（起 5 个 master 节点，分布在不同的机房尽量保证可用性）
>       a：得到当前的时间，微妙单位
>       b：尝试顺序地在 5 个实例上申请锁，当然需要使用相同的 key 和 random value，这里一个 client 需要合理设置与 master 节点沟通的 timeout 大小，避免长时间和一个 fail 了的节点浪费时间
>       c：当 client 在大于等于 3 个 master 上成功申请到锁的时候，且它会计算申请锁消耗了多少时间，这部分消耗的时间采用获得锁的当下时间减去第一步获得的时间戳得到，如果锁的持续时长（lock validity time）比流逝的时间多的话，那么锁就真正获取到了。
>       d：如果锁申请到了，那么锁真正的 lock validity time 应该是 origin（lock validity time） - 申请锁期间流逝的时间
>       e：如果 client 申请锁失败了，那么它就会在少部分申请成功锁的 master 节点上执行释放锁的操作，重置状态
>       
>   失败重试：
>       如果一个 client 申请锁失败了，那么它需要稍等一会在重试避免多个 client 同时申请锁的情况，最好的情况是一个 client 需要几乎同时向 5 个 master 发起锁申请。
>       另外就是如果 client 申请锁失败了它需要尽快在它曾经申请到锁的 master 上执行 unlock 操作，便于其他 client 获得这把锁，
>       避免这些锁过期造成的时间浪费，当然如果这时候网络分区使得 client 无法联系上这些 master，那么这种浪费就是不得不付出的代价了。
>   释放锁：
>       放锁操作很简单，就是依次释放所有节点上的锁就行了
>   ```
>
>   

###### 缓存异常

>   -   缓存雪崩（缓存同一时间大面积失效。造成短时间后端db承载大量请求而崩溃）
>       -   缓存数据的过期时间随机，防止同一时间大量数据过期现象发生
>       -   数据量请求不大时。加锁排队
>       -   缓存增加标记（是否失效），如果失效则更新缓存
>   -   缓存穿透（缓存和db中都没有数据，导致所有的请求落到db上。造成数据库短时间承载大量请求二崩溃）
>       -   接口层增加校验，如用户鉴权、基础校验（id合法性）等
>       -   无效数据缓存（缓存nil数据）
>       -   布隆过滤器。将所有可能存在的数据 hash 到一个足够大的bitmap中。一个一定不存在的数据会被 bitmap 拦截
>   -   缓存击穿（缓存失效时刻，大量读请求查询同一个key。）
>       -   热点数据不过期
>       -   锁
>   -   缓存预热
>   -   缓存降级
>   -   如何保证缓存和db数据的一致性



