---
layout: post
title:  "分布式 分库分表 ShardingSphereJDBC"
categories: 分布式
tags:  分布式 分库分表
author: 娄凯凯
---

* content
{:toc}

# ShardingSphere
ShardingSphere包含三个重要的产品，ShardingJDBC、ShardingProxy和
ShardingSidecar。其中sidecar是针对service mesh定位的一个分库分表插件，目
前在规划中。而我们今天学习的重点是ShardingSphere的JDBC和Proxy这两个组
件。其中，ShardingJDBC是用来做客户端分库分表的产品，而ShardingProxy是用
来做服务端分库分表的产品。这两者定位有什么区别呢？我们看下官方资料中给出
的两个重要的图





## ShardingJDBC
![ShardingJDBC](/image/shardShpere/shardJdbc.png)

shardingJDBC定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。它
使⽤客户端直连数据库，以 jar 包形式提供服务，⽆需额外部署和依赖，可理解为增
强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。

## ShardingProxy
![ShardingProxy](/image/shardShpere/ShardingProxy.png)

ShardingProxy定位为透明化的数据库代理端，提供封装了数据库⼆进制协议的服
务端版本，⽤于完成对异构语⾔的⽀持。⽬前提供 MySQL 和 PostgreSQL 版本，
它可以使⽤任何兼容 MySQL/PostgreSQL 协议的访问客⼾端。

## 那这两种方式有什么区别呢？

| | ShardingJDBC |ShardingProxy|
| ------- | ------- | ------- |
| 数据库   | 任意   | MySQL/PostgreSQL   |
|连接消耗数| 高    |低|
|异构语言 | 仅java |任意|
|性能 | 损耗低 | 损耗略高 |
|无中心化 |是 |否|
|静态入口 |无 | 有|

# ShardingJDBC实战
shardingjdbc的核心功能是数据分片和读写分离，通过ShardingJDBC，应用可以
透明的使用JDBC访问已经分库分表、读写分离的多个数据源，而不用关心数据源的
数量以及数据如何分布。

## 1、核心概念：
  - 逻辑表：水平拆分的数据库的相同逻辑和数据结构表的总称
  - 真实表：在分片的数据库中真实存在的物理表。
  - 数据节点：数据分片的最小单元。由数据源名称和数据表组成
  - 绑定表：分片规则一致的主表和子表。
  - 广播表：也叫公共表，指素有的分片数据源中都存在的表，表结构和表中的数据在每个数据库中都完全一致。例如字典表。
  - 分片键：用于分片的数据库字段，是将数据库(表)进行水平拆分的关键字段。SQL中若没有分片字段，将会执行全路由，性能会很差。
  - 分片算法：通过分片算法将数据进行分片，支持通过=、BETWEEN和IN分片。
  分片算法需要由应用开发者自行实现，可实现的灵活度非常高。
  - 分片策略：真正用于进行分片操作的是分片键+分片算法，也就是分片策略。在ShardingJDBC中一般采用基于Groovy表达式的inline分片策略，通过一个包含分片键的算法表达式来制定分片策略，如t_user_$->{u_id%8}标识根据u_id模8，分成8张表，表名称为t_user_0到t_user_7。 

## 简单配置文件说明
下面简单讲一下配置文件，详细的使用见配置文件


## ShardingSphere的分片算法
整个分库分表的核心就是在于配置的分片算法。
ShardingSphere目前提供了一共五种分片策略：
  - NoneShardingStrategy
    不分片。这种严格来说不算是一种分片策略了。只是ShardingSphere也提供了
这么一个配置。

  - InlineShardingStrategy 最常用的分片方式
    - 配置参数： inline.shardingColumn 分片键；inline.algorithmExpression
分片表达式
    - 实现方式： 按照分片表达式来进行分片。

  - StandardShardingStrategy
    只支持单分片键的标准分片策略。
    - 配置参数：standard.sharding-column 分片键；standard.precisealgorithm-class-name 精确分片算法类名；standard.range-algorithmclass-name 范围分片算法类名
    - 实现方式：
     shardingColumn指定分片算法。
     __preciseAlgorithmClassName__ 指向一个实现了io.shardingsphere.api.algorithm.sharding.standard.PreciseShardingAlgorithm接口的java类名，提供按照 = 或者 IN 逻辑的精确分片 示例：
       com.roy.shardingDemo.algorithm.MyPreciseShardingAlgorit
hm
    __rangeAlgorithmClassName__ 指向一个实现了io.shardingsphere.api.algorithm.sharding.standard.RangeShardingAlgorithm接口的java类名，提供按照Between 条件进行的范围分片。示例：
      com.roy.shardingDemo.algorithm.MyRangeShardingAlgorithm
    说明：其中精确分片算法是必须提供的，而范围分片算法则是可选的。
  - ComplexShardingStrategy
      支持多分片键的复杂分片策略。
      - 配置参数：complex.sharding-columns 分片键(多个);complex.algorithm-class-name 分片算法实现类。
      - 实现方式：  
        shardingColumn指  定多个分片列。
        algorithmClassName指向一个实现了org.apache.shardingsphere.api.sharding.complex.ComplexKeysShardingAlgorithm接口的java类名。提供按照多个分片列进行综合分片的算法。
        示例：
            com.roy.shardingDemo.algorithm.MyComplexKeysShardingAlgorithm
  - HintShardingStrategy
      不需要分片键的强制分片策略（强制路由）。这个分片策略，简单来理解就是说，他的分片键
      不再跟SQL语句相关联，而是用程序另行指定。对于一些复杂的情况，例如
      select count(*) from (select userid from t_user where userid in (1,3,5,7,9))
      这样的SQL语句，就没法通过SQL语句来指定一个分片键。这个时候就可以通过
      程序，给他另行执行一个分片键，例如在按userid奇偶分片的策略下，可以指定1作为分片键，然后自行指定他的分片策略。
      - 配置参数：hint.algorithm-class-name 分片算法实现类。
      - 实现方式：
        algorithmClassName指向一个实现了org.apache.shardingsphere.api.sharding.hint.HintShardingAlgorithm接口的java类名。 示例：
            com.roy.shardingDemo.algorithm.MyHintShardingAlgorithm
        在这个算法类中，同样是需要分片键的。而分片键的指定是通过HintManager.addDatabaseShardingValue方法(分库)和HintManager.addTableShardingValue(分表)来指定。使用时要注意，这个分片键是线程隔离的，只在当前线程有效，所以通常建议使用之后立即关闭，或者用try资源方式打开。而Hint分片策略并没有完全按照SQL解析树来构建分片策略，是绕开了SQL解析的，所有对某些比较复杂的语句，Hint分片策略性能有可能会比较好(情况太多了，无法一一分析)。但是要注意，Hint强制路由在使用时有非常多的限制：
        ```
        -- 不支持UNION
        SELECT * FROM t_order1 UNION SELECT * FROM t_order2
        INSERT INTO tbl_name (col1, col2, …) SELECT col1, col2, … FROM
        tbl_name WHERE col3 = ?

        -- 不支持多层子查询
        SELECT COUNT(*) FROM (SELECT * FROM t_order o WHERE o.id IN
        (SELECT id FROM t_order WHERE status = ?))

        -- 不支持函数计算。ShardingSphere只能通过SQL字面提取用于分片的值
        SELECT * FROM t_order WHERE to_date(create_time, 'yyyy-mm-dd') = '2019-01-01';
        ```
示例详见代码库application02.properties配置。
从这里也能看出，即便有了ShardingSphere框架，分库分表后对于SQL
语句的支持依然是非常脆弱的。

# ShardingSphere的SQL使用限制
参见官网文档： https://shardingsphere.apache.org/document/current/cn/fea
tures/sharding/use-norms/sql/ 文档中详细列出了非常多ShardingSphere目前版
本支持和不支持的SQL类型。这些东西要经常关注。

## 支持的SQL

|SQL                                                        |必要条件|
| ------- | ------- |
|SELECT * FROM tbl_name                                      |  |
|SELECT * FROM tbl_name WHERE (col1 = ? or col2 = ?)   and col3 = ? |      |
|SELECT * FROM tbl_name WHERE col1 = ? ORDER BY col2 DESC LIMIT ? |            |
|SELECT COUNT(*), SUM(col1), MIN(col1), MAX(col1), 
AVG(col1) FROM tbl_name WHERE col1 = ?             |   |
|SELECT COUNT(col1) FROM tbl_name WHERE col2 = ? 
GROUP BY col1 ORDER BY col3 DESC LIMIT ?, ?|               |
|INSERT INTO tbl_name (col1, col2,…) VALUES (?, ?, ….)| |
|INSERT INTO tbl_name VALUES (?, ?,….)| |
|INSERT INTO tbl_name (col1, col2, …) VALUES (?, ?, ….),(?, ?, ….)| |
|INSERT INTO tbl_name (col1, col2, …) SELECT col1,
col2, … FROM tbl_name WHERE col3 = ?   |  INSERT表和SELECT表必须为相同表或绑定表 |
|REPLACE INTO tbl_name (col1, col2, …) SELECT col1,
col2, … FROM tbl_name WHERE col3 = ? | REPLACE表和SELECT表必须为相同表或绑定表|
|UPDATE tbl_name SET col1 = ? WHERE col2 = ?| |
|DELETE FROM tbl_name WHERE col1 = ? | |
|CREATE TABLE tbl_name (col1 int, …) | |
|ALTER TABLE tbl_name ADD col1 varchar(10)| |
|DROP TABLE tbl_name|  |
|TRUNCATE TABLE tbl_name| |
|CREATE INDEX idx_name ON tbl_name| |
|DROP INDEX idx_name ON tbl_name | |
| DROP INDEX idx_name | |
|SELECT DISTINCT * FROM tbl_name WHERE col1 = ? | |
|SELECT COUNT(DISTINCT col1) FROM tbl_name| |
|SELECT subquery_alias.col1 FROM (select
tbl_name.col1 from tbl_name where tbl_name.col2=?)
subquery_alias | |

## 不支持的SQL

|SQL| 不支持原因|
| ------ | ------ |
|INSERT INTO tbl_name (col1, col2, …)
VALUES(1+2, ?, …)    |  VALUES语句不支持运算表达式 |
|INSERT INTO tbl_name (col1, col2, …) SELECT * FROM tbl_name WHERE col3
= ?| SELECT子句暂不支持使用*号简写及内 置的分布式主键生成器|
|REPLACE INTO tbl_name (col1, col2, …) SELECT * FROM tbl_name WHERE col3
= ? |SELECT子句暂不支持使用*号简写及内置的分布式主键生成器|
｜SELECT * FROM tbl_name1 UNION
SELECT * FROM tbl_name2｜ UNION ｜
|||
｜SELECT * FROM tbl_name1 UNION ALL
SELECT * FROM tbl_name2 ｜ UNION ALL ｜
|||
|SELECT SUM(DISTINCT col1),
SUM(col1) FROM tbl_name| 详见DISTINCT支持情况详细说明 |
|||
|SELECT * FROM tbl_name WHERE
to_date(create_time, ‘yyyy-mm-dd’)
= ?| 会导致全路由 |
|(SELECT * FROM tbl_name) | 暂不支持加括号的查询|
|||
|SELECT MAX(tbl_name.col1) FROMtbl_name|  查询列是函数表达式时,查询列前不能使
用表名;若查询表存在别名,则可使用表的别名|

## DISTINCT支持情况详细说明
### 支持的SQL

|SQL|
| ------ |
|SELECT DISTINCT * FROM tbl_name WHERE col1 = ?|
|SELECT DISTINCT col1 FROM tbl_name|
|SELECT DISTINCT col1, col2, col3 FROM tbl_name|
|SELECT DISTINCT col1 FROM tbl_name ORDER BY col1|
|SELECT DISTINCT col1 FROM tbl_name ORDER BY col2|
|SELECT DISTINCT(col1) FROM tbl_name|
|SELECT AVG(DISTINCT col1) FROM tbl_name|
|SELECT SUM(DISTINCT col1) FROM tbl_name|
|SELECT COUNT(DISTINCT col1) FROM tbl_name|
|SELECT COUNT(DISTINCT col1) FROM tbl_name GROUP BY col1|
|SELECT COUNT(DISTINCT col1 + col2) FROM tbl_name|
|SELECT COUNT(DISTINCT col1), SUM(DISTINCT col1) FROM tbl_name|
|SELECT COUNT(DISTINCT col1), col1 FROM tbl_name GROUP BY col1|
S|ELECT col1, COUNT(DISTINCT col1) FROM tbl_name GROUP BY col1|

### SQL 不支持原因

|SELECT SUM(DISTINCT tbl_name.col1), SUM(tbl_name.col1)FROM tbl_name|
查询列是函数表达式时,查询列前不能使用
表名;若查询表存在别名,则可使用表的别名|

# 分库分表带来的问题
  - 1、分库分表，其实围绕的都是一个核心问题，就是单机数据库容量的问题。我们
要了解，在面对这个问题时，解决方案是很多的，并不止分库分表这一种。但是
ShardingSphere的这种分库分表，是希望在软件层面对硬件资源进行管理，从而便
于对数据库的横向扩展，这无疑是成本很小的一种方式。
大家想想还有哪些比较好的解决方案？
  - 2、一般情况下，如果单机数据库容量撑不住了，应先从缓存技术着手降低对数据
库的访问压力。如果缓存使用过后，数据库访问量还是非常大，可以考虑数据库读
写分离策略。如果数据库压力依然非常大，且业务数据持续增长无法估量，最后才
考虑分库分表，单表拆分数据应控制在1000万以内。
当然，随着互联网技术的不断发展，处理海量数据的选择也越来越多。在实际进
行系统设计时，最好是用MySQL数据库只用来存储关系性较强的热点数据，而对海
量数据采取另外的一些分布式存储产品。例如PostGreSQL、VoltDB甚至HBase、
Hive、ES等这些大数据组件来存储。
  - 3、从上一部分ShardingJDBC的分片算法中我们可以看到，由于SQL语句的功能
实在太多太全面了，所以分库分表后，对SQL语句的支持，其实是步步为艰的，稍
不小心，就会造成SQL语句不支持、业务数据混乱等很多很多问题。所以，实际使
用时，我们会建议这个分库分表，能不用就尽量不要用。
如果要使用优先在OLTP场景下使用，优先解决大量数据下的查询速度问题。而在
OLAP场景中，通常涉及到非常多复杂的SQL，分库分表的限制就会更加明显。当
然，这也是ShardingSphere以后改进的一个方向。
  - 4、如果确定要使用分库分表，就应该在系统设计之初开始对业务数据的耦合程度
和使用情况进行考量，尽量控制业务SQL语句的使用范围，将数据库往简单的增删
改查的数据存储层方向进行弱化。并首先详细规划垂直拆分的策略，使数据层架构
清晰明了。而至于水平拆分，会给后期带来非常非常多的数据问题，所以应该谨
慎、谨慎再谨慎。一般也就在日志表、操作记录表等很少的一些边缘场景才偶尔用
用。

# 分库分表方案设计实战

接下来，我们来给电商的商品管理模块设计一个分库分表的方案，来理解下分库分表应该如何落地。
一个典型的电商场景，商品管理模块大致的功能组件如下图：

![tableStruct](/image/shardShpere/tableStruct.png)

针对这个场景，考虑到商品信息会持续增长，越来越多的情况，要如何设计分库
分表方案？
1、以业务为单位考虑对数据进行垂直分片，店铺、产品、商品三种业务数据垂直拆
分成三个不同的库。字典表作为广播表冗余到三个不同的库中。
2、考虑数据增长情况，商品将会是以后增长最快的数据，店铺和产品的数据增速会
逐渐降低。所以对商品表进行分片。
3、将关联性较强的商品信息表和商品补充信息表配置为绑定表。整体分库分表大致如下图：

![tableupdate](/image/shardShpere/tableStructUpdate.png)




