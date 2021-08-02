### 多版本并发控制(MVCC)

#### 0. 准备

每个 SQL 语句看到的都是数据库的一个快照，读写互不阻塞



![mvcc-defination](/home/zhngcho/Documents/comeaross-postgresql/figures/mvcc-defination.png)

#### 1. 数据库的隔离级别（Transaction Isolation）

- R

```shell
# 打开客户端
sudo su postgres -c psql
```



```sql
-- 建库
drop database if exists mydb;
create database mydb;
\c mydb
-- 1. 建表
drop table if exists tb_mvcc;
create table tb_mvcc
(
    id   int primary key,
    data int
);
-- 2. 插入 3 条数据
insert into tb_mvcc
select generate_series(1, 3), generate_series(1, 3);

```

t1-T1

```sql
begin;
select data from tb_mvcc where id = 3;
/*
 data 
------
    3
(1 row)
*/
```

t2-T2

```sql
begin;
update tb_mvcc set data = 4 where id = 3;
select data from tb_mvcc where id = 3;
/*
 data 
------
    4
(1 row)

*/
```

t3-T1

```sql
select data from tb_mvcc where id = 3;
commit;
/*
 data 
------
    3
(1 row)

*/
```

t4-T3

```sql
begin;
select data from tb_mvcc where id = 3;
/*
 data 
------
    3
(1 row)
*/
```

t5-T3

```sql
commit;
```

t6-T2

```sql
commit;
```

t7-T4

```sql
begin;
select data from tb_mvcc where id = 3;
/*
 data 
------
    4
(1 row)
*/
```

t8-T4

```sql
commit;
```



#### 1. PG 其实没有更新和删除，只有插入！





PG 真的没有删除？

```sql
show transaction_isolation;
/*
transaction_isolation 
-----------------------
 read committed
 (1 row)
 */
```

