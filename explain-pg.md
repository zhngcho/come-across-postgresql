### 看懂 postgres 的执行计划
[toc]
#### 环境准备
- ubuntu 20.04.1
- 安装数据库
    ```bash
    sudo apt install postgresql-12
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

#### 仅索引扫描

#### hash 连接

#### nestloop（嵌套循环） 连接
 1. 双层顺序扫描
 2. 内层 hash
 3. ...

#### merge 连接

#### bitmap 扫描

#### Append