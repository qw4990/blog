## INTRODUCTION

Q1

```sql
SELECT e1.employee_name, j.job_title
FROM employees e1, job_history j
WHERE 
	e1.emp_id = j.emp_id AND
 	j.start_date > '19980101' AND
 	e1.salary > 
		(SELECT AVG (e2.salary)
		FROM employees e2
 		WHERE e2.dept_id = e1.dept_id) AND 
 	e1.dept_id IN
 		(SELECT dept_id
 		FROM departments d, locations l
 		WHERE d.loc_id = l.loc_id AND l.country_id = 'US');
```

## TRANSFORMATION IN ORACLE

### 2.1 Heuristic Transformations

#### 2.1.1 Subquery Unnesting

规定没有去关联的子查询, 用TIS(apply)的方式执行. 

即把外部的每一行, 都带入子查询去执行一遍.



Oracle中大体有两种子查询去关联的方式.

1. 会生成inline view的方式, 其实就是把子查询简单转变后, 从where移动到select后, 然后和其他查询部分进行join.
2. 不会生成inline view, 能直接把子查询彻底转换为join, 甚至完全消除.

(在Oracle中, select后的子查询被称为inline view)

方式1为cost-based的, 方式2是直接执行的, 因为他总是能生成更优的执行计划.

方式2后面介绍, 下面是1个方式1的例子.

Q2

```sql
SELECT d.dept_name, d.budget
FROM departments d
WHERE EXISTS 
	(SELECT 1
 	FROM employees e
 	WHERE d.dept_id = e.dept_id
 	AND e.salary > 200000);
```

规定`employees.dept_id`是指向`departments.dept_id`的外键. 

则Q2可以被转换为semijoin, Q3

```sql
SELECT d.dept_name, d.budget
FROM departments d, employees e
WHERE d.dept_id = e.dept_id AND 
			e.salary > 200000;
```

为什么Q3在进行物理优化时, 会选择最优的执行方式.

而Q3的最差方式就是join顺序不变, 选择nested-loop join的方式.

这和转化前的apply等效, 所以转化一定会让执行计划变优.



Note: Semijoin is a noncommutative join and imposes a partial order on the joining tables.

(因为outer表的一行数据可能有多行inner表的数据与之对应， 所以我们不能交换他们的角色)

But we can convert this semijoin into an inner join by applying a sort distinct operator on the inner table.

outer semijoin inner ---> distinct(inner) join d;

#### 2.1.2 Join Elimination

根据表和列的性质, 去除join.

Q4

```sql
SELECT e.name, e.salary
FROM employees e, departments d
WHERE e.dept_id = d.dept_id; 
```

Q5

```sql
SELECT e.name, e.salary
FROM employees e left outer join 
departments d on e.dept_id = d.dept_id;
```

规定`employees.dept_id`是指向`departments.dept_id`的外键.

则Q4和Q5都可以被简化为Q6

```sql
SELECT e.name, e.salary
FROM employees e;
```

去除了多余的join操作, 一定会让执行计划变优.

#### 2.1.3 Filter Predicate Move Around

Q7

```sql
SELECT acct-id, time, ravg
FROM (SELECT acct-id, time, 
 AVG(balance) OVER (PBY acc-id OBY 
time 
 RANGE BETWEEN UNBOUNDED 
PROCEEDING 
 AND CURRENT ROW) ravg
 FROM accounts)
WHERE acct-id = ‘ORCL’ AND time <= 12;
```

Q8

```sql
SELECT acct-id, time, ravg
FROM (SELECT acct-id, time, 
 AVG(balance) OVER (PBY acc-id OBY 
 time RANGE BETWEEN UNBOUNDED
 PROCEEDING 
 AND CURRENT ROW) ravg
 FROM accounts
 WHERE acct-id = ’ORCL’ AND 
time<=12);
```

#### 2.1.4 Group Prunning

通过把其他部分和group key相关的过滤条件移动到agg内去, 减少group key的数量.

```sql
SELECT sum_sal, country_id, state_id, city_id
FROM (
  SELECT SUM (e.salary) sum_sal, 
 		l.country_id, l.state_id, l.city_id
 	FROM employees e,departments d, locations l
 	WHERE e.dept_id = d.dept_id AND 
  			d.location_id = l.location_id
	GROUP BY ROLLUP (country_id, state_id, city_id)
	)
WHERE city_id = ’San Francisco’;


SELECT sum_sal, country_id, state_id, city_id
FROM (
  SELECT SUM (e.salary) sum_sal, 
 		l.country_id, l.state_id, l.city_id
 	FROM employees e,departments d, locations l
 	WHERE e.dept_id = d.dept_id AND 
  			d.location_id = l.location_id AND
  			city_id = ’San Francisco’
	GROUP BY ROLLUP (country_id, state_id)
	);
```

外部的city_id过滤条件, 可以被移动到agg内, 减少group key.

减少了group的维度, 所以执行计划一定更优.

### 2.2 Cost-Based Transformations

#### 2.2.1 Subquery Unnesting

相当于把关联子查询从where移动到select后面, 并和外部其他部分进行join.

在移动过程中, 通常会把子查询根据关联列, 转变为agg或者distinct.

把整体的执行方式从apply转变为了join.

当然这个方式不一定优, 如果外部的选择率很高, 且子查询内有索引可以利用.

则apply的方式或许比join更高.

Transform Q1 to Q10

```sql
SELECT e1.employee_name, j.job_title
FROM employees e1, job_history j,
 (SELECT AVG(e2.salary) avg_sal, 
dept_id
 FROM employees e2
 GROUP BY dept_id) V
WHERE e1.emp_id = j.emp_id and
 j.start_date > '19980101' and
 e1.dept_id = V.dept_id and
 e1.salary > V.avg_sal and 
 e1.dept_id IN
 (SELECT dept_id
 FROM departments d, locations l
 WHERE d.loc_id = l.loc_id
 and l.country_id = 'US');
```

#### 2.2.2 Group-By and Distinct View Merging

相当于把agg或者distinct与外部部分合并, 在本例中, 把agg提到了join之上.

如果join的选择率比较高, 就能大量减少agg的数据量.

当然反之, 如果agg选择率比较高而join没那么高, 把agg放在join下应该是更好的.

Merging the group-by  view in Q10 can produce Q11

```sql
SELECT e1.employee_name, j.job_title
FROM employees e1, job_history j, 
 employees e2
WHERE e1.emp_id = j.emp_id and
 j.start_date > '19980101' and
 e2.dept_id = e1.dept_id and
 e1.dept_id IN
 (SELECT dept_id
 FROM departments d, locations l
 WHERE d.loc_id = l.loc_id
 and l.country_id = 'US')
GROUP BY e2.dept_id, e1.emp_id, j.rowid,
 e1.employee_name, j.job_title, 
 e1.salary
HAVING e1.salary > AVG (e2.salary);
```

#### 2.2.3 Join Predicate Pushdown

和`Subquery Unnesting`相反, 相当于把Join Key推进子查询内.

当外部的filter选择率较高或者子查询内可以利用索引时, 或许能更快.

另外该方法还有可能消除子查询内的group-by和distinct.

如下面Q12到Q13, 就把子查询内的distinct消除了.

Q12

```sql
SELECT e1.employee_name, j.job_title
	e2.employee_name as mgr_name
FROM employees e1, job_history j, employees e2,
	(SELECT DISTINCT dept_id
 	FROM departments d, locations l
 	WHERE d.loc_id = l.loc_id and
 	l.country_id IN ('UK','US')) V
WHERE e1.emp_id = j.emp_id AND
	j.start_date > '19980101' AND
	e1.mgr_id = e2.emp_id AND
	e1.dept_id = V.dept_id;
	
	
	
SELECT e1.employee_name, j.job_title
FROM employees e1, job_history j,
	(SELECT DISTINCT dept_id
 	FROM locations l
 	WHERE l.country_id IN ('UK','US')
  ) V
WHERE e1.emp_id = j.emp_id AND
	j.start_date > '19980101' AND
	e1.dept_id = V.dept_id;
	

SELECT e1.employee_name, j.job_title
FROM employees e1, job_history j,
	(SELECT dept_id
	FROM locations l
  WHERE l.dept_id = e1.dept_id AND
   l.country_id IN ('UK', 'US') 
  ) V
WHERE e1.emp_id = j.emp_id AND
	j.start_date > '19980101';
	
	
	
SELECT DV.employee_name, DV.job_title,
FROM (
	SELECT DISTINCT e1.emp_id, j.rowid,
  	e1.employee_name, j.job_title
	FROM employees e1, job_history j,
  	locations l
	WHERE l.country_id IN ('UK','US') AND
  	e1.emp_id = j.emp_id AND
    j.start_date > '19980101' AND
		e1.dept_id = l.dept_id) DV;
```

Q13

```sql
SELECT e1.employee_name, j.job_title
 e2.employee_name as mgr_name
FROM employees e1, job_history j, employees e2,
 (SELECT dept_id
 FROM departments d, locations l
 WHERE d.loc_id = l.loc_id and
 l.country_id IN (‘UK’,'US') and 
 e1.dept_id = d.dept_id) V
WHERE e1.emp_id = j.emp_id and
 j.start_date > '19980101' and
 e1.mgr_id = e2.emp_id;

```

#### 2.2.4 Group-By Placement

把group-by操作下推或者上拉.

比如把group-by下推到join之下, 或许可以大幅度减少join的数据量.

上拉也叫做`Group-By and Distinct View Merging`, 前面已经介绍过了.

#### 2.2.5 Join Factorization

Q14

```sql
SELECT e.first_name, e.last_name,
	e.job_id, d.department_name, l.city
FROM employees e, departments d,
     locations l
WHERE e.dept_id = d.dept_id AND
      d.location_id = l.location_id
UNION ALL
SELECT e.first_name, e.last_name,
j.job_id, d.department_name, l.city
FROM employees e, job_history j,
     departments d, locations l
WHERE e.emp_id = j.emp_id AND
      j.dept_id = d.dept_id AND
      d.location_id = l.location_id;
```

Q15

```sql
SELECT V.first_name, V.last_name,
	V.job_id, d.department_name, l.city
FROM departments d, locations l,
    (SELECT e.first_name, e.last_name,
            e.job_id, e.dept_id
     FROM employees e
     UNION ALL
     SELECT e.first_name, e.last_name,
             j.job_id, j.dept_id
     FROM employees e, job_history j
     WHERE e.emp_id = j.emp_id) V
WHERE d.dept_id = V.dept_id AND
      d.location_id = l.location_id;
```

#### 2.2.6 Predicate Pullup

把一些比较重的谓词上拉.

思路是可能外部有些条件可以帮助过滤掉不少数据.

上拉后, 可以减少这些较重谓词的数据量.

目前"比较重"的谓词定义为1)procedural languages, 2)user-defined operators, 3)subqueries.

目前只有当SQL中包含`rownum(limit)`时, 才会考虑这个规则.

Q16

```sql
SELECT *
FROM (SELECT document_id
 FROM product_docs
 WHERE 
	contains(summary,'optimizer',1) > 0 AND 
	contains(full_text,'execution',2) > 0
  ORDER BY create_date) V
WHERE rownum < 20;
```

考虑借住`rownum<20`来减少`contains`的数据量, 于是优化为Q17

```sql
SELECT *
FROM (SELECT document_id, value(r) AS vr
 FROM product_docs
 WHERE 
	contains(full_text,'execution',2)> 0
 ORDER BY create_date) V 
WHERE contains(summary, 'optimizer', 1) > 0 
AND rownum < 20;

```

大多数场景下谓词下推的效果应该更好, 而谓词上拉与之相反, 所以在运用该规则时需要借住代价估算来确保上拉后的效果.

#### 2.2.7 Set Operators Into Join

#### 2.2.8 Disjunction Into Union All

## FRAMEWORK FOR COST-BASED TRANSFORMATION

### 3.1 Basic Components

优化器会从底向上的优化query tree.

这里的`Physical Optimization`主要有两个用途:

1. 让`Cost-Based Transformation`查询某个query tree的代价;
2. 生成最后的物理执行计划;

在Oracle中优化规则的应用顺序如下:

```
common sub-expression factorization, SPJ view merging, join elimination, subquery unnesting, group-by (distinct) view merging, group pruning, predicate move around, set operator into join conversion, group-by placement, predicate pullup, join factorization, disjunction into union-all expansion, star transformation, and join predicate pushdown

```

当然也有不完全遵循这个顺序的时候.

### 3.2 State Space Search Techniques

假设某条query有N个objects(即能够被规则作用的单位, 如table, join edge, predicate等).

一条规则T, 能够作用在这N个objects上, 则我们有2^N种选择.

我们用长度为N的bit数组来表示这条query和这个规则的状态.

`bits[i]`为1或者0分别表示规则T是否对第i个object起了作用.

如果我们有M条规则, 则整个状态为一个M*N的bit举证.



为了解决搜索空间太大的问题, 参考了一系列随机搜索算法.

本质思想都是从搜索空间某一点开始, 找到一个局部最优解.

假设有N个objects和一条策略, 有四种搜索策略:

1. Exhaustive: 遍历所有的空间, 空间大小为2^N;
2. Iterative: 从起点开始, 一步一步向下找到一个局部最优解; 然后换一个起点, 继续找局部最优解; 指导不出现新的状态, 或步数到达某个阈值; 空间大小为N+1到2^N;
3. Linear: 基本思想是, 假设每个objects的优化是独立的, 然后利用动态规划找到最优解, 空间大小为N+1;
4. Two-pass: 只考虑两种情况, 要么把这条规则用到所有objects上, 要么一个都不用, 空间大小为2;

Oracle根据objects的数量来自动选择使用哪种搜索策略.

### 3.3 Interaction between Transformations

#### 3.3.1 Interleaving

如果有两条规则T1和T2, 当T1作用在某个object上后, T2也可以作用在object上.

则优化器会打乱上述的规则顺序, 立马把T2插到T1后执行.

可以避免一些因为搜索路径倾向于寻找局部最优而导致的bad case.



比如一开始在状态S1, 可以通过规则T1走到S2, 然后在S2通过T2走到S3.

如果S2的代价大于S1但是S3的代价小于S1和S2.

那么当处于S1时, 优化器会去尝试走到S2, 但是发现S2代价高于S1后.

则不会做T1的转换, 导致后续可能永远走不到S3上.

#### 3.3.2 Juxtaposition

如果有两条规则T1和T2, 都可以马上作用在同一个object上.

则优化器会打乱上述的规则顺序, 同时执行T1和T2, 比较他们的cost并选择执行哪条规则.

可以避免一些因为规则顺序不合理导致的bad case.



比如一开始在S1, 可以通过T1走到S2, S2代价略小于S1.

也可以通过T2从S1走到S3, S3代价远小于S1, 且T2排在T1之后.

在S1时, 如果不检查所有能作用的规则, 则可能因为先走到S2而永远走不到S3上.

#### 3.3.3 Impact on State Space

### 3.4 Optimization Performance

#### 3.4.1 Cost Cut-Off

如果某个状态的代价已经大于目前能找到的最优解, 则不会继续探索该状态.

#### 3.4.2 Reuse of Query Sub-Tree Cost Annotations

其实就是把已经优化过的子树的代价给记下来, 避免重复优化.

#### 3.4.3 Memory Management

纯物理层面, 就是把优化过程中的query tree等对象手动管理起来.

#### 3.4.4 Caching 

把某些较重的优化器操作进行缓存, 比如表的基数估计.



对应统计信息还没有被搜集的表, Oracle会采用动态采样的方式来预估该表的基数.

这个操作非常重, 于是在优化执行完毕后可以将这个数据缓存下来.

下次优化时则可以直接使用这个值.

### 4 PERFORMANCE STUDY

