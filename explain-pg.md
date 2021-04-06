### 看懂 postgres 的执行计划

[toc]

#### 环境准备

- ubuntu 20.04.1
- 安装数据库
  ```bash
  sudo apt install postgresql-12
  ```
- 启动 psql 客户端
  ```shell
  sudo su postgres -c psql
  ```
- 数据库初始化
  ```sql
  drop database if exists mydb;
  create database mydb;
  \c mydb
  ```

#### 全表扫描

##### 例 1. 无索引情况下全表扫描

```sql
-- 1. 建表
drop table if exists tb1;
create table tb1
(
    id   int primary key,
    data int
);
-- 2. 插入 10k 数据
insert into tb1
select generate_series(1, 10000), generate_series(1, 10000);
-- 3. 手动触发数据的抽样统计， 用于计划器生成更准确的执行计划
analyze;
-- 4. 查看建好的表
\d+ tb1

------------------查看执行计划-----------------
explain
select *
from tb1
where data < 8000;
/*
                    QUERY PLAN
--------------------------------------------------------
Seq Scan on tb1  (cost=0.00..170.00 rows=7999 width=8)
    Filter: (data < 8000)
(2 rows)

*/

-----------------名词解释----------------
/*
1. Seq Scan: 顺序扫描全表；
2. Filter: (data < 8000): 过滤器（表级过滤谓词）, 只会在读取所有元组的时候使用， 并不会减少需要扫描的表页面数量；
3. cost=0.00..170.00:
    （1）代表执行步骤的代价，单位并不是时间， 而是一个可配置的 benchmark；
    （2）0.00 代表启动代价，即读取到第一个元组之前花费的代价；
    （3）170.00 代表执行步骤的总代价；
4. rows=7999： 预计影响的行数；
5. width=8： 平均每行 8 字节；

*/

```

##### 例 2. 有索引的情况下全表扫描

```sql
explain
select *
from tb1
where id < 8000;
/*
                       QUERY PLAN
--------------------------------------------------------
 Seq Scan on tb1  (cost=0.00..170.00 rows=7999 width=8)
   Filter: (id < 8000)
(2 rows)

*/


```

#### 索引扫描

##### 例

```sql
explain
select *
from tb1
where id < 4000;
/*
                               QUERY PLAN
-------------------------------------------------------------------------
 Index Scan using tb1_pkey on tb1  (cost=0.29..139.27 rows=3999 width=8)
   Index Cond: (id < 4000)
(2 rows)

*/

-----------------名词解释----------------
/*
1. Index Scan: 索引扫描（索引是独立于数据的单独的文件）
2. using tb1_pkey on tb1: 使用的是 tb1 上的tb1_pkey 索引
*/

```

#### 仅索引扫描

##### 例

```sql
explain
select id
from tb1
where id < 4000;
/*
                                  QUERY PLAN
------------------------------------------------------------------------------
 Index Only Scan using tb1_pkey on tb1  (cost=0.29..139.27 rows=3999 width=4)
   Index Cond: (id < 4000)
(2 rows)
*/

-----------------名词解释----------------
/*
1. Index Only Scan： 仅索引扫描
2. 只需要扫描索引文件，不需要回表（访问数据文件）
*/
```

```sql
explain analyze
select id
from tb1
where id < 4000;
/*
                                                        QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using tb1_pkey on tb1  (cost=0.29..139.27 rows=3999 width=4) (actual time=0.055..1.620 rows=3999 loops=1)
   Index Cond: (id < 4000)
   Heap Fetches: 3999
 Planning Time: 0.159 ms
 Execution Time: 1.910 ms
(5 rows)
*/

-----------------名词解释----------------
/*
1. Heap Fetches: 与 MVCC 机制相关，当VM中的信息表明某页中可能存在不可见信息时，需要访问数据表中的可见信息
*/
```

#### 排序

##### 例 1

```sql
-- 1. 建表
drop table if exists tb2;
create table tb2
(
    id   int primary key,
    data int
);
-- 2. 插入 100 数据
insert into tb2
select generate_series(1, 100), generate_series(1, 100);
-- 3. 手动触发数据的抽样统计， 用于计划器生成更准确的执行计划
analyze;
-- 4. 查看建好的表
\d+ tb2
```

```sql
explain
select *
from tb2
where id < 200
order by id;
/*
                        QUERY PLAN
-----------------------------------------------------------
 Sort  (cost=5.57..5.82 rows=100 width=8)
   Sort Key: id
   ->  Seq Scan on tb2  (cost=0.00..2.25 rows=100 width=8)
         Filter: (id < 200)
(4 rows)
*/

-----------------名词解释----------------
/*
1. Sort: 排序操作，内存够用则快排，否则文件归并排序, 通常在 order by 或者 merge join 中出现
*/
```

##### 例 2

```sql
explain analyze
select *
from tb2
where id < 100
order by id;
/*
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------------
 Sort  (cost=5.53..5.78 rows=99 width=8) (actual time=0.103..0.125 rows=99 loops=1)
   Sort Key: id
   Sort Method: quicksort  Memory: 29kB
   ->  Seq Scan on tb2  (cost=0.00..2.25 rows=99 width=8) (actual time=0.023..0.062 rows=99 loops=1)
         Filter: (id < 100)
         Rows Removed by Filter: 1
 Planning Time: 0.205 ms
 Execution Time: 0.173 ms
(8 rows)
*/

-----------------名词解释----------------
/*
0. 机智的你发现这个执行计划里居然多了 actual xxx 的信息，没错，它就是真正的执行时间，这是因为我们在 explain 后边增加了 analyze 关键字，它会让后边的这个sql真正进行执行并记录每一步骤的执行时间，同时会有更丰富的输出信息
（注意，当执行update/delete/insert 语句时，这会导致数据受到影响，此时可以考虑先开启一个事务，分析完后再进行回滚）
1. Sort Method: 排序方法
2. Memory: 内存使用
3. Planning Time: 计划时间
4. Execution Time: 执行时间
5. Rows Removed by Filter: 被过滤条件移除的行数
6. loops: 循环次数
*/


```

#### hash 聚合

##### 例

```sql
explain
select sum(data)
from tb1
group by id;
/*
                          QUERY PLAN
---------------------------------------------------------------
 HashAggregate  (cost=195.00..295.00 rows=10000 width=12)
   Group Key: id
   ->  Seq Scan on tb1  (cost=0.00..145.00 rows=10000 width=8)
(3 rows)

*/

```

#### 分组聚合

```sql
explain
select sum(data)
from tb1
where id < 4000
group by id;
/*
                                  QUERY PLAN
-------------------------------------------------------------------------------
 GroupAggregate  (cost=0.29..199.25 rows=3999 width=12)
   Group Key: id
   ->  Index Scan using tb1_pkey on tb1  (cost=0.29..139.27 rows=3999 width=8)
         Index Cond: (id < 4000)
(4 rows)
*/

```

#### hash 连接

##### 例 1

```sql
explain
select a.id, b.id, a.data
from tb1 a,
     tb2 b
where a.data = b.data;
/*
                            QUERY PLAN
-------------------------------------------------------------------
 Hash Join  (cost=3.25..186.75 rows=100 width=12)
   Hash Cond: (a.data = b.data)
   ->  Seq Scan on tb1 a  (cost=0.00..145.00 rows=10000 width=8)
   ->  Hash  (cost=2.00..2.00 rows=100 width=8)
         ->  Seq Scan on tb2 b  (cost=0.00..2.00 rows=100 width=8)
(5 rows)
*/

-----------------名词解释----------------
/*
1. Hash Join: 哈希连接
2. Hash Cond: 哈希连接条件
3. Seq Scan ...: 顺序扫描 tb1 (外表)
4. Hash: 顺序扫描 tb2 并在内存建立 hash 表（内表）
5. 外表在上，内表在下
*/

```

##### 例 2

```sql
explain
select a.id, b.id, a.data
from tb1 a,
     tb2 b
where a.data = b.data
  and a.id < 100;
/*
                                  QUERY PLAN
------------------------------------------------------------------------------
 Hash Join  (cost=3.54..13.65 rows=1 width=12)
   Hash Cond: (a.data = b.data)
   ->  Index Scan using tb1_pkey on tb1 a  (cost=0.29..10.02 rows=99 width=8)
         Index Cond: (id < 100)
   ->  Hash  (cost=2.00..2.00 rows=100 width=8)
         ->  Seq Scan on tb2 b  (cost=0.00..2.00 rows=100 width=8)
(6 rows)
*/
```

##### 例 3

```sql
explain analyze
select a.id, b.id, a.data
from tb1 a,
     tb2 b
where a.data = b.data
  and a.id < 100;
/*
                                                       QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------
 Hash Join  (cost=3.54..13.65 rows=1 width=12) (actual time=0.126..0.230 rows=99 loops=1)
   Hash Cond: (a.data = b.data)
   ->  Index Scan using tb1_pkey on tb1 a  (cost=0.29..10.02 rows=99 width=8) (actual time=0.015..0.059 rows=99 loops=1)
         Index Cond: (id < 100)
   ->  Hash  (cost=2.00..2.00 rows=100 width=8) (actual time=0.065..0.066 rows=100 loops=1)
         Buckets: 1024  Batches: 1  Memory Usage: 12kB
         ->  Seq Scan on tb2 b  (cost=0.00..2.00 rows=100 width=8) (actual time=0.012..0.030 rows=100 loops=1)
 Planning Time: 0.279 ms
 Execution Time: 0.283 ms
(9 rows)
*/

-----------------名词解释----------------
/*
1. Buckets: hash 桶的数量
2. Batches: 批次（如果内存存不下整个的hash表，则把整个Hash表划分批次）
*/

```

#### merge 连接

##### 例

```sql
explain
select a.id, b.id, a.data
from tb1 a,
     tb2 b
where a.id = b.id
  and b.id < 10;
/*
                                    QUERY PLAN
----------------------------------------------------------------------------------
 Merge Join  (cost=2.66..6.21 rows=8 width=12)
   Merge Cond: (a.id = b.id)
   ->  Index Scan using tb1_pkey on tb1 a  (cost=0.29..318.29 rows=10000 width=8)
   ->  Sort  (cost=2.37..2.39 rows=8 width=4)
         Sort Key: b.id
         ->  Seq Scan on tb2 b  (cost=0.00..2.25 rows=8 width=4)
               Filter: (id < 10)
(7 rows)
*/

-----------------名词解释----------------
/*
1. Merge Join: 合并连接（类似归并排序的 merge 过程）
2. Merge Cond: 连接条件
3. Index Scan ... : 使用 tb1_pkey 索引进行扫描（扫描完有序）
4. Sort: 顺序扫描 tb2, 然后根据 id 排序
5. 合并排序需要被合并的两侧有序
*/
```

#### nested loop（嵌套循环） 连接

```sql
-- 1. 建表
drop table if exists tb3;
create table tb3
(
    id   int primary key,
    data int
);
-- 2. 插入 10 数据
insert into tb3
select generate_series(1, 10), generate_series(1, 10);
-- 3. 手动触发数据的抽样统计， 用于计划器生成更准确的执行计划
analyze;
-- 4. 查看建好的表
\d+ tb3
```

##### 例 1 物化顺序扫描嵌套循环连接

```sql
explain
select a.id, b.id, a.data
from tb1 a,
     tb2 b;
/*
                            QUERY PLAN
-------------------------------------------------------------------
 Nested Loop  (cost=0.00..12647.25 rows=1000000 width=12)
   ->  Seq Scan on tb1 a  (cost=0.00..145.00 rows=10000 width=8)
   ->  Materialize  (cost=0.00..2.50 rows=100 width=4)
         ->  Seq Scan on tb2 b  (cost=0.00..2.00 rows=100 width=4)
(4 rows)
*/

-----------------名词解释----------------
/*
1. Nested Loop: 嵌套循环连接
2. Materialize: 物化内表
*/
```

##### 例 2 嵌套循环连接的循环次数分析

```sql
explain analyze
select a.id, b.id, a.data
from tb1 a,
     tb2 b;
/*
                                                  QUERY PLAN
---------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.00..12647.25 rows=1000000 width=12) (actual time=0.050..132.123 rows=1000000 loops=1)
   ->  Seq Scan on tb1 a  (cost=0.00..145.00 rows=10000 width=8) (actual time=0.027..0.858 rows=10000 loops=1)
   ->  Materialize  (cost=0.00..2.50 rows=100 width=4) (actual time=0.000..0.004 rows=100 loops=10000)
         ->  Seq Scan on tb2 b  (cost=0.00..2.00 rows=100 width=4) (actual time=0.012..0.040 rows=100 loops=1)
 Planning Time: 0.137 ms
 Execution Time: 160.765 ms
(6 rows)

*/

-----------------名词解释----------------
/*
1. 注意loop
*/

```

##### 例 3 物化索引嵌套循环连接

```sql
explain
select a.id, b.id, a.data
from tb1 a,
     tb2 b
where a.id < 10;
/*
                                    QUERY PLAN
----------------------------------------------------------------------------------
 Nested Loop  (cost=0.29..21.71 rows=900 width=12)
   ->  Seq Scan on tb2 b  (cost=0.00..2.00 rows=100 width=4)
   ->  Materialize  (cost=0.29..8.49 rows=9 width=8)
         ->  Index Scan using tb1_pkey on tb1 a  (cost=0.29..8.44 rows=9 width=8)
               Index Cond: (id < 10)
(5 rows)
*/

```

#### bitmap 扫描

##### 例 1.

```sql
explain select * from tb1 where id < 100;
set enable_indexscan = false;
show enable_indexscan;
explain select * from tb1 where id < 100;
/*
                               QUERY PLAN                               
------------------------------------------------------------------------
 Bitmap Heap Scan on tb1  (cost=5.05..51.29 rows=99 width=8)
   Recheck Cond: (id < 100)
   ->  Bitmap Index Scan on tb1_pkey  (cost=0.00..5.03 rows=99 width=0)
         Index Cond: (id < 100)
(4 rows)

*/

-----------------名词解释----------------
/*
1. Bitmap: 位图 （优势是使用内存较小，而且对数据的随机读变为顺序读， 劣势是多一步形成位图的过程）
2. Recheck: 复核条件（位图扫描记录的是页，所以需要复核条件）
*/

```


##### 例 2.

```sql
-- 1. 建表
drop table if exists tb5;
create table tb5
(
    id   int primary key,
    data int
);
create index tb5_data_idx on tb5 using btree(data);
-- 2. 插入 10k 数据
insert into tb5
select generate_series(1, 10000), generate_series(1, 10000);
-- 3. 手动触发数据的抽样统计， 用于计划器生成更准确的执行计划
analyze;
-- 4. 查看建好的表
\d+ tb5

explain select * from tb5 where id < 100 and data < 50;
/*
                                    QUERY PLAN                                    
----------------------------------------------------------------------------------
 Bitmap Heap Scan on tb5  (cost=9.93..13.95 rows=1 width=8)
   Recheck Cond: ((data < 50) AND (id < 100))
   ->  BitmapAnd  (cost=9.93..9.93 rows=1 width=0)
         ->  Bitmap Index Scan on tb5_data_idx  (cost=0.00..4.65 rows=49 width=0)
               Index Cond: (data < 50)
         ->  Bitmap Index Scan on tb5_pkey  (cost=0.00..5.03 rows=99 width=0)
               Index Cond: (id < 100)
(7 rows)
*/

-----------------名词解释----------------
/*
1. BitmapAnd: 位图与（可以利用多个索引）
*/

```

##### 例 3.

```sql
explain analyze select * from tb1, tb5 where tb5.data < 100 and tb1.id = tb5.id order by tb1.data;
/*
                                                               QUERY PLAN                                                                
-----------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=227.07..227.32 rows=99 width=16) (actual time=4.874..4.882 rows=99 loops=1)
   Sort Key: tb1.data
   Sort Method: quicksort  Memory: 29kB
   ->  Hash Join  (cost=52.53..223.79 rows=99 width=16) (actual time=0.127..4.829 rows=99 loops=1)
         Hash Cond: (tb1.id = tb5.id)
         ->  Seq Scan on tb1  (cost=0.00..145.00 rows=10000 width=8) (actual time=0.019..2.257 rows=10000 loops=1)
         ->  Hash  (cost=51.29..51.29 rows=99 width=8) (actual time=0.091..0.092 rows=99 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 12kB
               ->  Bitmap Heap Scan on tb5  (cost=5.05..51.29 rows=99 width=8) (actual time=0.032..0.052 rows=99 loops=1)
                     Recheck Cond: (data < 100)
                     Heap Blocks: exact=1
                     ->  Bitmap Index Scan on tb5_data_idx  (cost=0.00..5.03 rows=99 width=0) (actual time=0.017..0.017 rows=99 loops=1)
                           Index Cond: (data < 100)
 Planning Time: 0.558 ms
 Execution Time: 4.938 ms
(15 rows)
*/

```

#### Append

通常出现在分区表中

```sql
set enable_indexscan = true;
show enable_indexscan;
-- 1. 建表
drop table if exists tb4;
create table tb4
(
    id   int primary key,
    data int
) partition by range (id);
create table tb4_0_100 partition of tb4 for values from (0) to (100);
create table tb4_100_200 partition of tb4 for values from (100) to (200);
create table tb4_200_300 partition of tb4 for values from (200) to (300);
-- 2. 插入数据
insert into tb4
select generate_series(1, 299), generate_series(1, 299);
-- 3. 查看建好的表
\d+ tb4

explain
select *
from tb4
where id > 90
  and id < 110;
/*
                                      QUERY PLAN
--------------------------------------------------------------------------------------
 Append  (cost=4.27..30.04 rows=22 width=8)
   ->  Bitmap Heap Scan on tb4_0_100  (cost=4.27..14.97 rows=11 width=8)
         Recheck Cond: ((id > 90) AND (id < 110))
         ->  Bitmap Index Scan on tb4_0_100_pkey  (cost=0.00..4.27 rows=11 width=0)
               Index Cond: ((id > 90) AND (id < 110))
   ->  Bitmap Heap Scan on tb4_100_200  (cost=4.27..14.97 rows=11 width=8)
         Recheck Cond: ((id > 90) AND (id < 110))
         ->  Bitmap Index Scan on tb4_100_200_pkey  (cost=0.00..4.27 rows=11 width=0)
               Index Cond: ((id > 90) AND (id < 110))
(9 rows)
*/

-- 4. 手动触发数据的抽样统计， 用于计划器生成更准确的执行计划
analyze;

explain
select *
from tb4
where id = 90
   or id = 110;

/*
                           QUERY PLAN
-----------------------------------------------------------------
 Append  (cost=0.00..5.00 rows=4 width=8)
   ->  Seq Scan on tb4_0_100  (cost=0.00..2.48 rows=2 width=8)
         Filter: ((id = 90) OR (id = 110))
   ->  Seq Scan on tb4_100_200  (cost=0.00..2.50 rows=2 width=8)
         Filter: ((id = 90) OR (id = 110))
(5 rows)

*/
```

#### 并行查询

```sql
-- 1. 建表
drop table if exists tb6;
create table tb6
(
    id   int primary key,
    data int
);
-- 2. 插入 10m 数据
insert into tb6
select generate_series(1, 10000000), generate_series(1, 10000000);
-- 3. 手动触发数据的抽样统计， 用于计划器生成更准确的执行计划
analyze;
-- 4. 查看建好的表
\d+ tb6

explain analyze                                     
select sum(data)
from tb6;
/*
                                                                QUERY PLAN                                                                
------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=97331.43..97331.44 rows=1 width=8) (actual time=429.057..450.167 rows=1 loops=1)
   ->  Gather  (cost=97331.21..97331.42 rows=2 width=8) (actual time=428.950..450.159 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=96331.21..96331.22 rows=1 width=8) (actual time=407.071..407.078 rows=1 loops=3)
               ->  Parallel Seq Scan on tb6  (cost=0.00..85914.57 rows=4166657 width=4) (actual time=0.036..236.370 rows=3333333 loops=3)
 Planning Time: 0.273 ms
 Execution Time: 450.219 ms
(8 rows)
*/

-----------------名词解释----------------
/*
1. Gather: 把所有结果收集起来
2. Partial Aggregate: 部分聚合
3. Finalize Aggregate: 最终聚合
4. Parrallel Seq Scan: 并行顺序扫描
*/

```

#### 实例分析

#### 参考资料

- [PG 官方文档](https://www.postgresql.org/docs/12/using-explain.html)
- PostgreSQL 修炼之道：从小工到专家 (作者：唐成)
- PostgreSQL 即学即用 第 3 版 数据库 （瑞金娜 奥贝 等 著，丁奇鹏 译 ）
- PostgreSQL 指南内幕探索 (作者:(日)Hironobu Suzuki(铃木启修))
