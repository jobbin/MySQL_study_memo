# MySQL_study_memo

## [01 | 基础架构：一条SQL查询语句是如何执行的？](https://time.geekbang.org/column/article/68319)

- 存储引擎层
  - 负责数据的存储和提取
  - 存储引擎插件式: InnoDB、MyISAM、Memory等  
  ※ MySQL v5.5.5起以InnoDB为默认存储引擎

![architecure_of_MySQL](images/architecure_of_MySQL.png)

 - 连接器
   - 参数 wait_timeout(8h,默认值)
   - 长连接
     - MySQL执行时使用的内存会在连接断开时释放，长连接过久有可能导致内存消耗大
     - 推荐定期断开长连接
     - (>= MySQL v5.7)执行 mysql_reset_connection 来重新初始化连接资源，无需重连和权限验证

- 查询缓存 (MySQL 8.0开始无此功能)
  - 只要有对一个表的 **更新**，这个表上 **所有的查询缓存都会被清空**
  - 长时间才会更新的 **静态表** 比较推荐使用缓存
  - query_cache_type → DEMAND : 默认的 SQL 语句都不使用查询缓存

- 分析器
  - **识别** MySQL语句里面的字符串分别是什么，代表什么
  - **分析** MySQL语法

- 优化器
  - 多个索引的时候，决定使用哪个索引；
  - 在一个语句有多表关联（join）的时候，决定各个表的连接顺序

- 执行器
  - 判断执行查询的权限
  - 根据表的引擎定义，使用引擎提供的接口

## [02 | 日志系统：一条SQL更新语句是如何执行的？](https://time.geekbang.org/column/article/68633)

[Write-Ahead Logging](https://ja.wikipedia.org/wiki/%E3%83%AD%E3%82%B0%E5%85%88%E8%A1%8C%E6%9B%B8%E3%81%8D%E8%BE%BC%E3%81%BF): **先写日志**，再写磁盘

- redo log
  - InnoDB特有日志
  - 当一条记录需要更新时，InnoDB引擎会先把记录写到redo log里，然后更新内存，更新完成；InnoDB会在适当的时候将这个操作记录更新写入磁盘。
  - redo log大小是固定的，使用如下图，从头开始写，写到末尾就又回到开头**循环写**  
    write pos 是当前记录的位置  
    checkpoint 是当前要擦除的位置  
    write pos 追上 checkpoint 时，这时候不能再执行新的更新，得停下来先擦掉一些记录
  - **crash-safe**: 保证即使数据库发生异常重启，之前提交的记录都不会丢失
  ![redo_log](images/redo_log.png)


- bin log 和 redo log的区别
  - bin log是server层的日志，所以引擎都可以用；redo log是InnoDB引擎特有的。
  - redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
  - redo log 是循环写的，空间固定会用完； binlog 是可以追加写入的。


```
mysql> create table T(ID int primary key, c int);

mysql> update T set c=c+1 where ID=2;
```
Update流程例子  
redo log 的写入拆成了两个步骤：prepare 和commit，这就是"**两阶段提交**"  
保证数据库的状态和用它的日志恢复出来的库的状态保持一致
![update_flow_sample](images/update_flow_sample.png)

## [03 | 事务隔离：为什么你改了我还看不见？](https://time.geekbang.org/column/article/68963)

- SQL 标准的事务(transaction)隔离级别  
**※show variables 可查看参数 transaction-isolation 的值**

  1. 读未提交（read uncommitted）
    - 一个事务还没提交(commit)时，它做的变更就能被别的事务看到
  2. 读提交（read committed
    - 一个事务提交之后，它做的变更才会被其他事务看到
  3. 可重复读（repeatable read）
    - 一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的
    - 未提交变更对其他事务也是不可见的
  4. 串行化（serializable ）
    - 对于同一行记录，“写”会加“写锁”，“读”会加“读锁”
    - 当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行

- MySQL 的事务启动方式有以下两种
  1. begin 或 start transaction ,配套的提交语句是 commit，回滚语句是 rollback
  2. set autocommit=0，这个命令会将这个线程的自动提交关掉
    - **注意**: 有些客户端连接框架会默认连接成功后先执行一个 set autocommit=0 的命令。
    - 建议使用 set autocommit=1 和 [commit work and chain](https://dev.mysql.com/doc/refman/5.6/ja/commit.html)（提交事务后自动启动下一个事务)

- **查询长事务**
  - 示例：用information_schema 库的 **innodb_trx** 查找持续时间超过 60s 的事务
```
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

## [04 | 深入浅出索引（上）](https://time.geekbang.org/column/article/69236)

InnoDB 使用 **B+ 树** 索引模型，数据都存储在 B+ 树中的

- 每一个索引在 InnoDB 里面对应一棵 B+ 树
  - 主键索引(**clustered index**)的叶子节点存的是 **整行数据**
  - 非主键索引(**secondary index**)的叶子节点内容是 **主键的值**


- 示例
```
mysql> create table T(
id int primary key,
k int not null,
name varchar(16),
index (k))engine=InnoDB;
```
表中 R1~R5 的 (ID,k) 值分别为 (100,1、(200,2)、(300,3)、(500,5) 和 (600,6)
![innoDB_b+_tree](images/innoDB_b+_tree.png)


- 基于主键索引和普通索引的查询的区别
  - select * from T where ID=500, 即主键查询方式，则只需要搜索 ID 这棵 B+ 树
  - select * from T where k=5，即普通索引查询方式，则需要先搜索 k 索引树得到 ID 的值为 500，再到 ID 索引树搜索一次。这个过程称为 **回表**


InnoDB 的 [B+树](https://techlife.cookpad.com/entry/2017/04/18/092524)

- 参考资料
  - [B TreeとB+ Treeの違い](https://christina04.hatenablog.com/entry/2017/05/17/190000)
  - [ヤフー社内でやってるMySQLチューニングセミナー大公開](https://www.slideshare.net/techblogyahoo/mysql-58540246)

## [05 | 深入浅出索引（下）](https://time.geekbang.org/column/article/69636)

- 覆盖索引: (查询字段已经在索引里)无需 **回表** 的索引
- 联合索引: 根据创建联合索引的顺序，以最左原则进行where检索
  - 最左前缀 → 联合索引的最左 N 个字段，也可以是字符串索引的最左 M 个字符
- 索引下推:
  - MySQL 5.6 引入的索引下推优化（index condition pushdown）
  - 可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数

## [06 | 全局锁和表锁 ：给表加个字段怎么有这么多阻碍？](https://time.geekbang.org/column/article/69862)

MySQL里面的锁: 全局锁，表级锁，行锁

- 全局锁
  - 对真个数据库加锁，命令是 Flush tables with read lock(FTWRL)
  - 典型使用场景是，做全库逻辑备份

官方逻辑备份工具 mysqldump,当使用 **参数–single-transaction** 的时候，导数据之前就会启动一个事务，来确保拿到 **一致性视图**。由于 **MVCC** 的支持, 这个过程中**数据是可以正常更新**

- 表级锁(2种)
  - 表锁
    - 语法: lock tables … read/write
    - 释放锁的方式： unlock tables主动释放；客户端断开时自动释放
    - 不仅影响其他线程，也影响自己
  - 元数据锁(meta data lock: MDL)
    - MySQL v5.5时引入MDL
    - (系统默认自动加)当对一个表做增删改查操作的时候，加 MDL 读锁；当要对表做结构变更操作的时候，加 MDL 写锁
    - **读锁之间不互斥**，因此你可以有多个线程同时对一张表增删改查
    - **读写锁之间、写锁之间是互斥的**


![table_lock_problem](images/table_lock_problem.png)
- sessionA 执行 **begin**, select语句时自动加读锁
- sessionB 执行select语句时自动加读锁，读锁之间不互斥，没有影响
- sessionC 执行alter语句时加写锁，因与sessionA，sessionB的读锁互斥，需等待sessionA，sessionB解锁
- sessionD 执行select语句时自动加读锁，因与sessionC的写锁互斥，需等待sessionC解锁
- **★ 如果该表查询语句频繁，且客户端有重试机制的话，这个数据库的线程很快就会爆满**
  - ★★避免该问题的方法： **在 alter table 语句里面设定等待时间**








.
