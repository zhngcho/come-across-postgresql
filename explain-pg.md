### 看懂 postgres 的执行计划
[toc]
#### 环境准备

#### 全表扫描

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