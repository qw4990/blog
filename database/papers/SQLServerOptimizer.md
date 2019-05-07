## Introduction to Query Optimization

SQL Server Database Engine包含两个主要组件: 

1. storage engine,
2. query processor.



query processor的过程有:

1. Parsing: SQL到语法树;
2. Binding: 把语法树的节点和DB内部结构绑定, 比如table, column等, 输出结构为代数树;
3. Query Optimization: 根据代数树生成执行计划;
4. Query Execution: 根据执行计划执行并返回结果.



优化器两个核心挑战:

1. 搜索空间可能很大, 优化器不可能遍历所有状态, 因此需要保证在能遍历到的状态中, 包含有低代价(较优)的执行计划;
2. 精确的代价和基数估算.



Join Order是优化器中最复杂的问题之一, 两个核心的挑战在于:

1. join顺序的选择;
2. join实现算法的选择.

Join优化的结果可能有Left-Deep, Right-Deep, Bushy-Tree等倾向.

其中Left-Deep, Right-Deep的空间大小和表数量n的关系为: n!.

而Bushy-Tree为(2n-2)!/(n-1)!.

这还是只考虑了顺序, 而不考虑算法实现选择.



## The Execution Engine

### Data Access Operators

定义了三种数据结构, 两种访问操作.

三种数据结构为:

1. Heap: 包含了所有列的数据, 无序保存;
2. Clustered index: 包含了所有列的数据, 以key的顺序保存;
3. Non-clustered index: 包含了部分列的数据, 以key的顺序保存;

两种访问操作和他们的关系如下:

| Structure           | Scan                 | Seek                 |
| ------------------- | -------------------- | :------------------- |
| Heap                | Table Scan           |                      |
| Clustered index     | Clustered Index Scan | Clustered Index Scan |
| Non-clustered index | Index Scan           | Index Scan           |



还有一种访问操作为 bookmark lookup.

就是先通过某一个non-clustered index读取到一些信息, 但是这些信息不能完全包含用户所需.

再通过其他方式, 如走clustered index去拿出额外的信息.

### Aggregations

两种聚合实现:

1. Stream Aggregate: 两个输出必须有序;
2. Hash Aggregate: 把小表当做inner, 构建hash表, outer去probe;

### Joins

三种Join实现:

1. Nested Loops Join;
2. Merge Join;
3. Hash Join: 如果没有足够的内存构建hash表, 可以使用磁盘空间;

### Parallelism

串行的方案不能保证就一定比并行的方案好, 最后会把他们一起比较执行代价.

并行化通过专用的并行化的物理算子实现(exchange operator), 也就是:

1. Distribute Stream;
2. Gather Stream;
3. Repartition Stream;

三个逻辑算子的实现.

## Statistics and Cost Estimation

### Creating and updating statistics

三个最主要的统计信息为:

1. histogram;
2. the density informa;
3. the string statistics;



通过列的修改量计数(colmodctrs), 来衡量统计信息是否过期.

默认假设列之间是相互独立的, 如果有问题可以使用多列的索引.

### Inspecting statistics objects

### Density

Density就是`1/number of distinct values`, 就是基数的倒数.

### Histogram

在构建histogram的过程中, 使用了`maxdiff`算法.

Histogram把压缩后的数据信息保存在多个step(bucket)中.

每个step包含:

1. RANGE_HI_KEY: 上界;
2. RANGE_ROWS: step内除开上界的行数;
3. EQ_ROWS: step内值等于RANGE_HI_KEY的行数;
4. DISTINCT_RANGE_ROWS: step的基数;
5. AVG_RANGE_ROWS: 就是(RANGE_ROWS+EQ_ROWS)/DISTINCT_RANGE_ROWS;

### Statistics Maintenance

### Statistics on Computed Columns

当统计信息失效时, 会使用`30%`作为默认的选择率.

对于多列的运算, 可以使用Computed Columns Statistics, 比如:

```sql
ALTER TABLE Sales.SalesOrderDetail ADD cc AS OrderQty * UnitPrice
```

### Filtered Statistics

就是在统计信息上加filter过滤掉一些行, 比如:

```sql
CREATE STATISTICS california 
ON Person.Address(City) 
WHERE StateProvinceID = 9
```

可以用来处理一些列之间数据不独立, 导致统计信息预估不准的情况.

### Cardinality Estimation Errors

### Update Statistics with Rowcount, Pagecount

### Cost Estimation

Microsoft没有公开SQL Server的cost model.

在旧版的SQL Server中, cost的单位是预估的执行时间.

在较新的版本中, 表示一个单位的度量, 其本质上可以代表任何信息, 比如:

1. 执行时间;
2. CPU资源;
3. 网络资源;
4. ...

SQL Server在计算I/O资源时, 是以page为单位的.

## Index Selection

### Introduction

SQL Server能够同时利用多个索引, 来覆盖查询需要的数据.

因为bookmark lookup需要random I/O, 所以他的执行代价会比较高.

### The Mechanics of Index Selection

利用索引的`leading key`, 结合查询中的条件, 可以进行seek操作, 比如:

1. ProductID = 771;
2. UnitPrice < 3.975;
3. LastName = 'Allen';
4. LastName LIKE 'Brown%';

如果条件包含复杂的表达式, 如函数, 则就不行了:

1. ABS(ProductID) = 771;
2. UnitPrice + 1 < 3.975;
3. LastName LIKE '%Allen';
4. UPPER(LastName) = 'Allen';

对于包含多个列的索引, SQL Server能够利用前导列去进行seek, 如:

1. ProductID = 771 AND SalesOrderID > 34000;
2. LastName = 'Smith' AND FirstName = 'Lan';

如果前导列不能被进行seek, 即使后面的列满足条件也不行, 如:

1. ABS(ProductID) = 771 AND SalesOrderID = 34000;
2. LastName LIKE '%Smith' AND FirstName = 'Lan';

### The Database Engine Tuning Advisor

### The Missing Indexes Feature

### Unused Indexes

## The Optimization Process

优化器的目的不是找到最好的执行计划, 而是在搜索空间中找到一个足够好的执行计划即可.

### Overview

1. Simplification;
2. Quality Trivial Plan? Yes: end,No: 3;
3. Qualify Phase 0? Yes: 3,No: 6;
4. Phase 0;
5. Cost < Phase 0 Threshold? Yes: end,No: 6;
6. Phase 1;
7. Cost < Phase 1 Threshold? Yes: end,No: 8;
8. Qualify Parallel Plan? Yes: 9, No: 10;
9. Phase 1 Parallel;
10. Phase 2;

### Peeking at the Query Optimizer

在`sys.dm_exec_query_transformation_stats`中放了目前支持的所有转化规则.

其中有一个字段为`promise information`告诉优化器这条规则多有用.

不过并没有提及`promise information`的计算方法, 且看起来其值和query无关, 是全局静态的.

### Parsing and Binding

### Transformation Rules

转换规则用来把算子树(operator tree)转化成等价的另一棵树.

所有生成的结构都存放在内存一个叫`memo`(来自cascade framework)的结构中.

规则被分为三种类型:

1. simplification: 用于简化, 如Join Elimination;
2. exploration: 逻辑上进行搜索空间的扩展, 如Join Reorder;
3. implementation: 逻辑算子到物理算子, 如Join选择;

### The Memo

Memo用来存储搜索时产生的所有的结构和数据.

Memo包含多个group, 每个group内都包含逻辑上等价的算子树.

Memo被设计来避免重复优化, 逻辑等价的结构会被放在同一个group中, 只会被优化一次.



开始时优化器会把初始算子树的每个算子都放入一个group, 然后把这些group放入memo.

接着进行优化, 过程中利用转化规则产生新的结构和数据.

算子之间的指向, 是直接指向对应的group, 而不是对应group内的某个算子或树.

### Optimization Phases

整个阶段包括:

1. simplification;
2. trivial plan optimization;
3. full optimization:
   1. search 0;
   2. search 1;
   3. search 2.

### Simplification

在真正优化前, 用来简化, 常见的规则有:

1. 子查询转化为join;
2. Join消除;
3. Where条件下推.

### Trival Plan

有些特别简单的SQL, 比如`SELECT * FROM XXTable`, 是不需要进行优化过程的.

这个阶段就是判断查询是不是属于这种特别简单的SQL, 如果是则跳过后面的优化.

### Full Optimization

Full optimization又分多个阶段, 如果每个阶段产生的执行计划已经比较好, 则可以直接返回.

##### Search 0

又叫`transaction processing phase`.

主要是为一些`transaction processing system`中的小query进行优化, 这些query通常比较简单, 但是却又包含多个表(至少3个).

这个阶段会用一些启发式的规则, 进行`join reorder`.

如果最后的执行计划代价小于某个阈值, 则直接返回, 否则进入下一阶段.

##### Search 1

又叫`quick plan`.

利用一些更复杂一些的启发式规则, 来进一步优化.

如果结果比较好, 则直接返回.

否则会尝试将query进行并行化优化.

最后会把所有生成的执行计划, 并行化或者串行化的, 都拿来比较, 选一个最优.

如果最优的代价小于某个阈值, 则直接返回, 否则进入下一阶段.

##### Search 2

又叫`full optimization`.

也就是利用所有的转化规则, 在memo空间中进行搜索.

在搜索过程中, 会有一个动态的`timeout`, 用来终止搜索.
