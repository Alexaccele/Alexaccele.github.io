---
title: MySQL性能优化——剖析单条查询
date: 2019/02/17 16:04:35
tags: [MySQL,性能优化]
cagetories: MySQL
---
# MySQL性能优化——剖析单条查询

## EXPLAIN 的使用

EXPLAIN的使用非常简单，就在你要进行的查询语句前跟上 EXPLAIN 即可。
例如：
<!--more-->

    mysql> EXPLAIN SELECT * FROM house limit 5\G;
    *************************** 1. row ***************************
       			id: 1
       select_type: SIMPLE
    		 table: house
        partitions: NULL
              type: ALL
     possible_keys: NULL
               key: NULL
           key_len: NULL
               ref: NULL
              rows: 9
          filtered: 100.00
             Extra: NULL
    1 row in set, 1 warning (0.06 sec)

或者去掉\G

    mysql> EXPLAIN SELECT * FROM house limit 5;
    +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
    | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
    +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
    |  1 | SIMPLE      | house | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    9 |   100.00 | NULL  |
    +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)

各列的含义如下:

- id: SELECT 查询的标识符. 每个 SELECT 都会自动分配一个唯一的标识符.
- select_type: SELECT 查询的类型.
- table: 查询的是哪个表
- partitions: 匹配的分区
- type: join 类型
- possible_keys: 此次查询中可能选用的索引
- key: 此次查询中确切使用到的索引.
- ref: 哪个字段或常数与 key 一起被使用
- rows: 显示此查询一共扫描了多少行. 这个是一个估计值.
- filtered: 表示此查询条件所过滤的数据的百分比
- extra: 额外的信息


## 使用 SHOW PROFILE
开启show profile

`mysql> SET profiling = 1;`

任意执行查询语句

例如， `mysql> SELECT * FROM house limit 5；`

之后再执行show profiles;
可以查看结果

    mysql> show profiles;
    +----------+------------+-----------------------------+
    | Query_ID | Duration   | Query                       |
    +----------+------------+-----------------------------+
    |        1 | 0.00078075 | SELECT * FROM house limit 5 |
    +----------+------------+-----------------------------+
    1 row in set, 1 warning (0.00 sec)

其中duration则是执行单条指令的查询响应时间。

通过指定profile的查询ID可以查看更加具体的信息

    mysql> show profile for query 1;
    +----------------------+----------+
    | Status               | Duration |
    +----------------------+----------+
    | starting             | 0.000167 |
    | checking permissions | 0.000020 |
    | Opening tables       | 0.000079 |
    | init                 | 0.000099 |
    | System lock          | 0.000026 |
    | optimizing           | 0.000010 |
    | statistics           | 0.000030 |
    | preparing            | 0.000023 |
    | executing            | 0.000003 |
    | Sending data         | 0.000138 |
    | end                  | 0.000011 |
    | query end            | 0.000022 |
    | closing tables       | 0.000026 |
    | freeing items        | 0.000108 |
    | cleaning up          | 0.000021 |
    +----------------------+----------+
    15 rows in set, 1 warning (0.03 sec)

使用show profile命令所得到的执行时间更加精确，这对我们分析查询语句并对其进行优化很有帮助。
