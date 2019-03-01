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
