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
##### 例1. 无索引情况下全表扫描
```sql
-- 1. 建表
drop table if exists tb1;
create table tb1(
    id      int primary key, 
    data    int
);
-- 2. 插入 10k 数据
insert into tb1 select generate_series(1, 10000), generate_series(1, 10000);
-- 3. 手动触发数据的抽样统计， 用于计划器生成更准确的执行计划
analyze;
-- 4. 查看建好的表
\d+ tb1

------------------查看执行计划-----------------
explain select * from tb1 where data < 8000;
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
##### 例2. 有索引的情况下全表扫描
```sql
explain select * from tb1 where id < 8000;
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
explain select * from tb1 where id < 4000;
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
explain select id from tb1 where id < 4000;
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

#### 排序
##### 例 1
```sql
explain select * from tb2 where id < 200 order by id;
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
explain analyze select * from tb2 where id < 100 order by id;
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
explain select sum(data) from tb1 group by id;
/*
                          QUERY PLAN                           
---------------------------------------------------------------
 HashAggregate  (cost=195.00..295.00 rows=10000 width=12)
   Group Key: id
   ->  Seq Scan on tb1  (cost=0.00..145.00 rows=10000 width=8)
(3 rows)

*/

```

### 分组聚合
```sql
explain select sum(data) from tb1 where id < 4000 group by id;
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
```sql
-- 1. 建表
drop table if exists tb2;
create table tb2(
    id      int primary key, 
    data    int
);
-- 2. 插入 100 数据
insert into tb2 select generate_series(1, 100), generate_series(1, 100);
-- 3. 手动触发数据的抽样统计， 用于计划器生成更准确的执行计划
analyze;
-- 4. 查看建好的表
\d+ tb2
```
##### 例 1
```sql
explain select a.id, b.id, a.data from tb1 a, tb2 b where a.data = b.data;
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
3. Seq Scan ...: 顺序扫描 tb2 (外表)
4. Hash: 顺序扫描 tb1 并在内存建立 hash 表（内表）
5. 外表在上，内表在下
*/

```
##### 例 2
```sql
explain select a.id, b.id, a.data from tb1 a, tb2 b where a.data = b.data and a.id < 100;
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
explain analyze select a.id, b.id, a.data from tb1 a, tb2 b where a.data = b.data and a.id < 100;
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
explain select a.id, b.id, a.data from tb1 a, tb2 b where a.id = b.id and b.id < 10;
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
create table tb3(
    id      int primary key, 
    data    int
);
-- 2. 插入 10 数据
insert into tb3 select generate_series(1, 10), generate_series(1, 10);
-- 3. 手动触发数据的抽样统计， 用于计划器生成更准确的执行计划
analyze;
-- 4. 查看建好的表
\d+ tb3
```
##### 例 1 物化顺序扫描嵌套循环连接
```sql
explain select a.id, b.id, a.data from tb1 a, tb2 b;
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
explain analyze select a.id, b.id, a.data from tb1 a, tb2 b;
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
explain select a.id, b.id, a.data from tb1 a, tb2 b where  a.id < 10;
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

#### Append

#### 实例分析