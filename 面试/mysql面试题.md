##### mysql 常见面试题

- b树和b+树有什么区别，为什么Mysql使用B+树？

  > 

- innodb中b+树是怎么产生的？

- 高度为3的b+树可以存多少条数据？

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

- mysql锁有哪些？如何理解？

  > 

- Mysql 是如何保证数据不丢的？

  > WAL（Write-Ahead Logging） 机制。预写日志系统
  >
  > 只要redo log 和 binlog 能保证持久化磁盘。就可以确保Mysql 异常重启后，数据可以恢复。
  >
  > So：如何保证redo log 和 binlog 一定持久化磁盘
  >
  > redolog 写入机制？
  >
  > 
  >
  > binlog 写入机制？
  >
  > 

- Mysql 是如何保证主备一致性的？

  > 主备同步的基本原理？
  >
  > 

- Mysql 是如何保证高可用的？

  - 