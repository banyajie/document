#### Redis



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

###### RDB

>   快照
>
>   原理
>
>   
>
>   -   优点
>
>       -   只有一个RDB文件，方便持久化
>       -   性能最大化，独立子进程处理，不影响主进程数据处理
>       -   数据量大时，比AOF效率高
>
>   -   缺点
>
>       -   因为是定时同步，所以有丢失数据的可能
>
>       

###### AOF（append only file）

>   



##### Redis 集群方案

###### Redis Cluster

>   

###### Codis

>   

###### Twemproxy

>   



##### 常见面试题

###### 分布式锁

###### 排行榜

###### 数据一致性

###### 缓存穿透、缓存击穿

###### 雪崩



