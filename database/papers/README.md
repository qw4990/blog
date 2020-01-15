## 论文简结

### Volcano-An Extensible and Parallel Query Evaluation System
看得比较快, 主要引入了Open-Next-Close的volcano模型.


### Access Path Selection in a Relational Database Management System
看了前4节, 主要引入了AcessPath和Cost-Based, 在多条AcessPath上计算代价, 选取最优解.
在计算代价的部分, 利用简单的统计信息, 提出了一些简单的算法.
不过代价计算算法和其存储引擎比较绑定, 且对数据假设太过简单.


### MonetDB/X100: Hyper-Pipelining Query Execution
```
Section 2简述CPU工作原理, 点出对性能影响比较大的几个因素:
1. cpu cache
2. branch predict
3. loop pipeline
其中提到了几个比较有意思的数据:
1. 30%的指令为内存读写
2. 内存读写需要几十ns, 需要CPU等待几百个cycle

Section 3分析了TPC-H Q1在Mysql和MIL运行情况
在Mysql上面表现不佳原因:
1. 90%时间都处理都在做一些其他(如行解析)之类的工作
2. 实际处理时, 每次只处理一行, 不能利用cpu loop pipeline
在同为列存的MIL上表现不佳的原因:
1. MIL实现原因, 物化太多, 导致了内存瓶颈

Section 4简述了X100架构
1. 在MIL的基础上, 增加了selection-vector, 减少物化
2. 针对agg/join等算子进行了优化

Section 5比较了vector size的影响
1. size为1时, 相当于按行处理, 性能最差
2. size约等于L1和L2 Cache和时, 性能最好
3. size超过L1+L2时, 性能逐渐下降

其他:
物化这个概念比较有意思, 任何关于数据的拷贝都可以算作是物化
1. 数据从Disk读到Mem可以算作一次物化
2. MIL处理vector时的中间结果, 也可以当做在内存上进行了一次物化
3. Cache满时, 回写Mem, 也可看做物化
```

### Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age
```
介绍了一种在NUMA架构下的执行器框架.
主要思想为:
1. 精心设计每个算子的执行逻辑, 让他们都能被拆成多个子任务, 并发的执行, 利用多核的计算资源.
2. 在调度子任务时, 考虑数据的分布, 尽量把任务分配到离数据近的核上(核和数据在同一个socket上).

Morsel指的是每个子任务对应的输入数据.
Morsel-Driven指的就是以这些子任务极其数据为单位进行调度的执行框架.

Section 2以3个表的join举例描述Morsel-Driven的执行过程.

Section 3描述调度器的实现.
大致就是为每个Query拆分后的子任务, 维护一个列表, 每个子任务当前在哪个核上运行.
当某个核心的子任务运行结束, 调用dispatcher的代码申请新的子任务运行.
其中涉及到一些细节, 如优先级, 数据分布考虑, 并发限制, morsel大小考虑等不在这里赘述.

Section 4描述了各个算子的实现.
4.2小节引入了一个 lock-free tagged hash table, 大概思想为:
一个普通的hash散列表, 为每个散列表, 附加上一个小的bloom-filter.
其他几小节分布描述了Join, Agg, Sort的并行化实现, 很容易想到, 就不在赘述.

Section 5和6没细看.

总体看Morsel-Driven最大优点是把一个大Query拆分成多个子任务, 充分利用超多核的计算资源.
子任务调度时考虑内存分布对计算进行提速.
并且调度是在runtime进行, 所以能在运行过程中较为弹性的进行调度, 如某个任务数据较大, 就把他切分细一点, 多分配些计算资源.
```

### The Volcano Optimizer Generator: Extensibility and Efficient Search
```
Section 2 阐述了optimizer的设计目标和输入输出.
parser首先query转化为logical tree, 然后提供给optimizer进行优化, 产出一个执行计划.
引入了这么几个概念:
1. transformation rule: 逻辑算子的转化规则, 如InnerJoin(1, 2)可以转换为InnerJoin(2, 1).
2. implemention rule: 逻辑算子到物理实现的规则, 如InnerJoin可被转成InnerHashJoin或者InnerNLJoin, 就可以表示为两条规则.
3. a set of algorithms: 这里的algorithms其实就是物理算子的具体实现, 如InnerHashJoin/InnerNLJoin, 同时还包含其Cost信息.
4. physical property: 对物理实现的一些属性要求, 如order等.
5. enforcer: 是一种物理算子, 用来强制实现一些property, 如某个算子的结果不能有序, 但外部又希望他的输出有序, 则可以在他的头上加一个enforcer.

Section 3 阐述了整个搜寻最优解的过程.
类似于记忆化搜索, 输入为: (LogicalExpr, PhysProp, Limit), 表示搜寻改LogicalExpr在PhysProp限制下, cost不超过Limit的最优解.
搜寻过程大致为, 不断的利用 transformation rule和implemention rule和enforcer 扩展搜索空间并找到最优解.
transformation rule和implemention rule都是针对局部的转化, 对于一些比较全局的规则, 比如 predicates pushdown, 可以在执行搜索前, 执行一个preprocess的过程进行处理.
```


### Orca: A Modular Query Optimizer Architecture for Big Data
```
Section 3 讲整体结构, 由于Orca是独立于Database的单独的优化器, 因此定义了一个DXL语言来交换信息.
内部结构方面, 较为重要的是memo这个结构, 源自cascade framework.
1. 每个query都会对应一个memo, memo表示这个query的整个执行计划搜寻空间.
2. memo包含多个group, 多个group之间有依赖关系, 每个group表示"一部分逻辑的搜索空间", 如"Sel->Scan".
3. group包含多个group expression, 表示该种group的实现方式, 如"Selection->IndexScan"或者"Selection->TableScan"等.

group可以用来剪枝已经搜索过的逻辑.
目前的TiDB只能记忆化某棵子树的最优解, 而不能记忆化如搜索树中部某块局部逻辑.
比如某条分支路径为"Proj->Sel->Proj->Scan", 通过group, 就可以单独记住中间"Sel->Proj"的最优执行计划.
当以后在其他分支再遇到相同的逻辑时, 则可以跳过搜索.

Section 4.1 讲搜寻最优解的过程, 主要有这几个阶段:
1. Exploration: 根据规则扩展group或者group expression, 主要是逻辑上的扩展, 比如InnerJoin(1,2)可扩展为InnerJoin(2,1).
2. Statistics Derivation: 导出统计信息, 必要时会通过DXL从Database上获取必要信息.
3. Implemention: 根据规则导出group expression的实现, 如从InnerJoin导出InnerNLJoin/InnerHashJoin.
4. Optimization: 在搜寻空间中查找最优解, 类似于一个记忆化搜索的过程.

上述规则大概可以分为三种类型:
1. transformation rules: logical -> logical.
2. implemention rules: logical -> physical.
3. enforce rule: deal with enforced property.

由于上述规则一般都是针对局部的优化, 对于一些比较全局的规则, 比如 predicates pushdown, 可以在执行搜索前, 执行一个preprocess的过程进行处理.

Section 4.2 讲并发的在搜寻空间中搜索, 较为简单.

Section 6.1 提到一个用来复现bug的工具 AMPERe.

Section 6.2 提到一个用来调整cost model的工具 TAQO.
```


### Efficiently Compiling Efficient Query Plans for Modern Hardware
```
Section 1 讲了传统iterator(Volcano) model的缺点:
1. 一次只能处理一个tuple.
2. 通常用函数指针等方式实现, 效率比较差.
3. 额外的状态记录开销, 比如TableScan, 需要记录当前位置等状态.

通过batch的方式可以提高改模型的效率, 但是会导致另外一个问题.
Batch后会破坏pipeline的性质.
pipeline性质通常指的是不用拷贝的直接将数据传递给父节点.
在本文更为严格, 指的是数据不能出寄存器, 其实也就是在寄存器和内存间拷贝数据.
因为batch后, 通常数据量会变大, 寄存器不能存下.

Section 3 提出了新的模型
首先严格的定义了pipeline性质, 希望被处理的数据, 尽量留在寄存器中, 把多个算子的逻辑一起处理完后再拿出来.
作者认为所有 iterator-style + pull 的模式都会增加破坏pipeline性质的风险, 因此把数据流的方式从pull变成了push.

为此, 为每个算子提供了两个接口抽象:
1. produce(): 通知这个算子可以开始生产数据了.
2. consume(attributes, source): 供儿子节点调用, 用于让儿子把他数据push给父亲.

上述两个接口只是在代码生成时使用, 生成为实际的机器码后, 这个结构将不复存在, 变为更加紧致的整体.

Section 4 讲了生成机器码的方式
一开始想生成C++的手写代码, 但是编译太慢, 复杂的query编译可能就需要几秒钟.
后面改成了C++和LLVM协同的方式.

Section 5 讲了怎么并发
没看
```

### Large-scale Incremental Processing Using Distributed Transactions and Notifications
```
Section 1 交代了一些背景

Section 2.1 因为Percolator建立在Bigtable上, 这一节交代了Bigtable提供的接口.
主要其实就是自带时间戳和行级事务.

Section 2.2 讲解了跨行事务的实现.
整体方法还是2PC, 特色是利用了Bigtable的单行事务, 具体的细节就不描述了, 简单说明下该方法的可行性:
1. 如果事务在primary行提交前或primary行提交时失败: 那么所有行的write列还未改变, 读取时会直接读事务之前最新的版本数据, 相当于该事务对数据无任何影响.
2. 如果在primary行提交成功后, secondaries提交失败: primary已经成功则无任何影响, secondaries处于未完成状态, 当某个事务读取secondaries数据时, 发现其处于未完成状态, 则去检测它对于的primary的状态, 便可以区分它之前所处的事务是成功还是失败, 然后进行相对应的清理操作, rollforward或者rollback.
相当于整个事务的成功和失败状态由primary保证, 而primary的一致性由Bigtable自带单行事务保证, secondaries的一致性由Percolator上述的lazy cleanup完成.
两者一起, 则保证了整个事务的可靠性.
不过如果secondaries在清理状态的时候还失败, 且一直失败...

Section 2.3 讲了TSO的高吞吐实现.
也很简单的, 每次申请一个区间, 然后把区间最高值持久化到磁盘.
处理请求时从则从内存里面已经提前申请好的区间里面分配.
如果crash了, 则从持久层的上次区间最高值恢复.
```

### Linearizability: A Correctness Condition for Concurrent Objects
```
Section 1 非正式的, 从直觉上, 介绍了linearizability是什么样的一个模型.
在物理时间(绝对时间)上, 每个操作都有开始和结束时间, 分别定义为inv和res.
每个操作的生效时间, 可以是inv到res中的任一时刻.

假设有下面的3个操作者, P1, P2, P3, 分别把x的值设置为1, 把x的值设置为2, 读取x的值.

P1和P2的时间有相交, 比如顺序为inv1, inv2, res1, res2, inv3, res3.
则P1和P2的生效时间可以任意, 此时P3读取到x为1或者2都是合法的.
因为可以假设生效顺序为: P1, P2, P3或者P2, P1, P3.
这个可以当做规则1, 把所有操作转化成了全局的原子操作, 且P的内部顺序得到保证(这点在该例子中没有体现).

如果P1的结束事件在P2开始前, 执行顺序为inv1, res1, inv2, res2, inv3, res3.
则P1必须先生效, 然后P2生效, 且P3一定看到的值是2.
此时只能是P1, P2, P3.
这是规则2, 相当于保留了物理时间下的绝对偏序.

规则1把相交的操作转换为不相交, 得到一个全局的原子操作顺序, 如[inv1, inv2, res1, res2]和[inv1, inv2, res2, res1];
根据e1和e2的生效时间, 可以转换为[inv1, res1, inv2, res2]或者[inv2, res2, inv1, res1], 分别相当于;
[x=1, x=2]或者[x=2, x=1]两个写操作原子性的执行.
当然对于任何的P, 他们内部顺序在全局的顺序中必须得到保持, 比如P4的两个操作为, 读取x, 读取y.
则在全局视角的操作中, 也必须为P4先读取了x, 再读取y.

规则2则是在1的基础上, 保留了物理时间下的偏序.
和sequential consistency比较, 相当于多了规则2的限制.
Spanner论文中提出了external consistency, 其实就是linearizability, 只不过这里的偏序是以物理时间为参照, Spanner中是以TrueTime作为参照.

Section 2 严格的定义了linearizability, 定义过程中引入了一些概念.
invocation & response: 对应操作的开始和结束, 每个事件还有对应的发起者, 在论文中表示为Processor P, 和操作对象object x.
History: 由上述两种事件构成的序列.
Sequential History: 由相邻且对应的inv和res事件构成的history, 如[inv1, res1, inv2, res2]就是sequential的, 而[inv1, inv2, res1, res2]就不是, 该性质相对于可以把每个操作都看做原子性的, 中间没有被其他操作间断.
Equivalent History: 只要两个History关于每个Processor的子历史H|P相等, 则他们两个相等.
偏序<H: 对于两个事件e0, e1, 如果res(e0)发生在(物理时间)inv(e1)前, 则e0 <H e1; 该定义表示e0和e1的执行事件没有相交部分, 且e0绝对的发生在e1前.

Linearizable History: 只要能找到一个Sequential History S, 使得H == S, 且H当中的所有偏序关系<H, 在S中都保留, 则H是Linearizable的.
其中
两个条件其实就是Section 1中的两个规则.

Section 3 讲了linearizability的一些性质, 以及和其他模型的对比.
local property: H是linearizable的, 当且仅当对于H中的每个对象x, H|x是linearizable.
nonblocking property: 有点没理解... 

同Sequential consistency对比: Sequential consistency相当于要求所有Processor有一个全局同一的执行顺序视角就OK, 相当于只有规则1没有规则2.
比如P1, P2, P3分别设置x为1, 设置x为2, 读取x.
操作顺序为: [inv1, res1, inv2, res2, inv3, res3].
如果P3读取到x为1, 在Sequential consistency的定义中也是合法的, 相当于生效顺序为P2, P1, P3.
而在Linearizability中就是非法的.

Section 4及后续没看
```

### Spanner: Google’s Globally Distributed Database
```
看了Section 1, 2, 3, 4, 其中重点看了事务模型部分, 既Section 3, 4;
由于分析Spanner的博客太多了, 就不做过度赘述了, 推荐以下两个博客;
http://loopjump.com/google_spanner/
https://www.cnblogs.com/foxmailed/p/3867814.html

这里简单概括一下Spanner的事务模型.

Spanner提供的一致性模型为external consistency: 指的是后开始的事务一定能看到已经提交事务的结果.
实际上这就是linearizability, 只不过在linearizability中的时间标准是real-time(真实物理时间), 而在Spanner中是true-time(记为tt), 既由其原子钟和GPS形成的.

引入原子钟+GPS构成true-time, 就是为了给整个集群提供了一个时间标准, 既如果在某台机器上tt.After(xx)为true, 那么在整个集群上, xx时间点在逻辑上也一定都被认为"已经过去", 既tt.After(xx)也为true.

实现external consistency的核心方法有两步, 1)选择合适的commit时间戳, 记为ts, 2)等到在true-time时间标准上, ts已经"过去"后, 再提交, 也就是tt.After(ts)为true.

PS: 下面所说的所有关于时间的比较, 先后, 大小于, 都是以tt为标准在描述, a在b后发生表示a.After(b)为true.

第一步选择的ts一定要大于事务的开始时间, 另外要大于2PC中所有参与者的prepare时间, 另外还要保证生成的ts一定要比前一个事务的ts大.

第二步保证, 当tt.After(ts)为true后, 一旦事务提交, 它的结果便能对所有节点可见; 因为既然在这台机器上tt.After(ts)为true, 那么在所有简单, tt.After(ts)也一定为true.
```

### Online, Asynchronous Schema Change in F1
```
Section 2介绍了一些背景
F1底层为KV存储, F1会把KV转化成关系型的表示.
具体的存储方式如下:
设表为t, 有k1, k2两个primary key, a, b两个其他列.
则存在:
1. t.k1+t.k2+"exists"构成的key, 其value为null, 表示该行的存在.
2. t.k1+t.k2+"a"构成的key, 其value为t.a, 存储该行a列的内容.
3. t.b同上.

Section 3描述了其模型, 是最为核心的章节.
基本思路为: 作者认为数据库在状态变更时出现不一致的核心原因是变化太快导致的("The fundamental cause of this corruption is that the change made to the schema is, in some sense, too abrupt."), 于是引入了中间状态来解决这个问题.

3.1 节定义了引入的几个状态, absent, delete-only, write-only, public.

3.2 节对文中描述的一致性进行了定义, 简单来说就是不存在以下两种异常:
1. 多数据: 如某个索引上有某行的数据, 但是在实际的表中, 却没有对应的行.
2. 缺数据: 某行数据真实存在, 但是在索引上却找不到他对应的记录.

举例说明一个多数据异常发生的情况.
以添加索引为例, 把schema从S1状态变更为S2, 由于集群状态不同步, 假设机器A处于S1, 机器B处于S2.
B插入某行, 由于索引在S2上存在(public), 索引B会更新索引, 接着A删除该行, 由于索引在S1不存在(absent), A会留下索引上对应的行数据, 出现多数据异常.

3.3 节介绍了改变schema的过程.
为了简化, 论文假设任意时刻, 整个集群最多只存在相邻的两种状态的schema.
同时把schema的变更分为了optional和required两种模式.
区别在于optional的索引或列, 允许索引缺行, 或者列为null, 而required则不允许.
更实质的影响在于在整个schema的变更过程中, 是否会有一个时间段来重组织数据(reorg).
比如增加一个required索引, 则需要在某个时间段, 把添加索引前的所有数据, 都更新到索引上(backfill).

如果为optional, 则变更过程为:
absent -> delete-only -> public.

如果为required, 则过程为:
absent -> delete-only -> write-only -> public
多出的write-only实际上就是为的reorg阶段服务的.

证明的过程其实也简单, 以optional为例, 只要分别证明:
1. S1和S2分别在absent, delete-only不会引入异常, 且
2. S1和S2分别在delete-only, public不会引入异常.
则可以证明整个变更过程不会引入异常.

4 节介绍了实现上的一些点
4.2 说到每次变更效率较低(如reorg), 可把多个DDL操作打包一起执行.
4.3 上述有假设"任意时刻, 整个集群最多存在两种相邻的schema", 这是通过租约实现的, 每台机器定期从一个固定的KV存储里, 获取最新的schema状态, 获取失败节点则自杀, 等待外部集群管理系统处理.
```

### Cost-Based Query Transformation in Oracle
详细见[CBQT](./CBQT.md)

### Inside the SQL Server Query Optimizer
详细见[SQL Server Optimizer](./SQLServerOptimizer.md)

### C-Store: A Column-oriented DBMS
```
section1交代背景和C-Store基本架构;

总的来说C-Store适合处理大量大查询+小TP写入的场景;

主体分为两个大模块, Writeable Store(WS)和Read-optimized Store(RS);

RS用来支持大数据下的大查询, 列存;

WS假设数据规模远小于RS, 需要较多内存使用, 用来支持小TP写入, 存储格式和RS一样;

后台有Tuple Mover, 会将WS的数据移动到RS.



section2描述数据模型;

把表按列拆分成多个Projection, 每个Projection包含多个列, 并按照sort key(某一列)有序;

同一个列可能出现在多个Projection内, 也就是数据有冗余;

其他表的列也可能以外键的形式出现在Projection中;

如某个表有(name, age, dept, salary)四个列, 则可能被拆成:

1. (name, age | age);
2. (dept, age, ForeignKey | ForeignKey);
3. (name, salary | salary);

竖线后为sort key;

每个Projection会按照sort key, 水平切割成多个segment, 每个segment保存一段区间数据;



接着引入一个storage key的概念, 其实就是逻辑上的rowID; 

在RS中, storage key不会显示记录, 而是根据偏移量来决定;

为了把多个Projection逻辑上能还原成一个表, 引入join index的概念;

join index也是按Projection的方式进行存储, 包含两列(segmentID, storage key);



section3介绍了RS;

主要介绍了RS中的4中压缩方式;

用哪种压缩方式, 是根据order key和列基数确定的;

1. self-order, 小基: (v, f, n), v是value, f是v开始的偏移量, n是出现次数;
2. foreign-order, 小基: (v, b), v是value, b是bitstring, 表示出现在哪些行;
3. self-order, 大基: 把每个v和前一个v做差, 如(1, 4, 7, 7)会被压缩成(1, 3, 3, 0);
4. foreign-order, 大基: 不进行压缩;



section4介绍了WS;

数据模型和RS一样, 也是用RS;

犹豫假设数据量远小于RS, 不会对WS中的数据进行压缩;

且会利用B-tree来维护sort key和storage key;



section6介绍了更新和事务;

实现了一个简单版的snapshot isolation;

只读事务的时间戳必须要在一个区间[low water mark(LWM), high water mark(HWM)]内, 原因和实现有关;

WS内有insertion vector(IV)和deleted recored vector(DRV), 用来暂存一段时间内的插入和删除操作;

会把整个时间分为多个epoch, 有一个节点被用作timestamp authority(TA), 用来推进epoch;

TA定期的向其他节点发送end of epoch信号, 其他节点等该epoch开始的写入事务结束后, 返回信息;

TA收到所有的回馈信息后, 把epoch推进一个周期, 然后把HWM设置为上一个epoch.

我理解这样做的原因是:

1. C-Store预设场景决定, 大查询+小更新, 这种简单的SI实现已经够用;
2. 如论文中所说"without requiring a large space budget", 减小空间负担, WS多在内存中;



section7介绍Tuple Mover;

一个后台任务会寻找worthy segment, 然后更新到RS, 同时更新LWM;



section8简单介绍了执行器和优化器;

执行器比较有特点的是:

1. Select返回的是bitstring, 而不是直接过滤后的Projection;
2. 新增Mask算子, 接受bitstring和Projection, 返回对应过滤后的Projection;
3. 对应的bitstring算子, 直接在bitstring上进行操作;

优化器的主要任务有两个:

1. 决定用哪些projection来产生数据;
2. 决定什么时候用mask来解开bitstring, 这对效率影响很大, 很多操作and, or, count等可以直接在bitstring上进行;



section9用TPCH和传统的行存和其他的列存进行了对比;

总结起来速度快的原因又:

1. 列存, 可以避免读取无用的列数据;
2. 以Projection的方式存储数据, 不同Projection的sort order可以不一样;
3. 更好的数据压缩方式;
4. 部分算子操作可以直接在压缩后的数据上进行;
```


### Mesa: Geo-Replicated, Near Real-Time, Scalable Data Warehousing
```
section1介绍mesa的需要解决的问题和基本思路, 大概如下:

1. storage scalability and availability: 数据水平切分, 然后分组和备份.
2. consistent and repeatable queries during update: MVCC.
3. update scalability: batch.
4. update consistency across multiple data centers: Paxos.



section2主要介绍了数据模型.

首先把数据维度分为attributes(keys)和measure(values).

每个table的schema包括keys, values和聚合函数F, F必须满足结合律.

一个table可以包含一个或多个index.

更新时会把数据做batch, 然后分配一个version.

定期把数据进行预聚合, 形成的数据组成为delta.

每个delta包含一段version的数据, 表示为[v1, v2]&v1<=v2.

每次更新形成的delta成为singleton, 表示为[v1, v2]&v1=v2.

根据一些压缩策略, 把delta进行合并, 大概会形成:

[0, B], [B+1, v1], [v1, v2]...这样的形式.



delta是不可修改的, 是逻辑概念, 按照key排序.

存储时水平切割为多个data file, 每个file内再水平切割并组织成多个row block.

每个block会表示内部第一行的first key, block内按列存储, 以提高压缩率.

每个data file都会有一个index file, 包含了data file内每个row block的first key的前缀.

搜索时先在index file上根据前缀二分, 然后再在data file内进行二分找到需要的block.



section3讲Mesa架构.

大体包含两个子系统, update/maintenance(UMS)和querying(QS).

Mesa的原信息存储依赖BigTable, data file存储依赖Colossus.



先从单个实例的角度来看.

UMS是一个多worker的框架, 大体为一个Controller管控多个其他worker.

状态都存在BigTable中, 所以UM是无状态的.

QS接受用户请求, 然后从Colossus中读取对应的数据, 计算并返回.

不同请求有不同优先级, 优先级需要用户自己进行标记.



集群视角来看.

有一个Mesa committer组件, 接管所有更新请求, 他过程如下:

1. batch这些请求;
2. 申请version并对修改操作的元信息进行广播(如新数据的路径);
3. 等待一定数量的controller都完成对应广播元信息操作;
4. 把数据写到colossus;
5. 更新改表对应的version, 使得对之后的query生效.

这里广播和version管理都是通过另一个组件versions database完成的.

versions database利用Paxos, 可以保持多机房原信息一致性.

这里有个问题是为什么不把元信息的存储从Bigtable整合到versions database内?

目前猜测是元信息并不需要version那么强的一致性, Paxos会让元信息更新变得很慢.



以集群视角来看.

有一个global locator service组件, mesa client在发起请求前, 会访问他以获得一些query server的信息.



section4 讲了一些优化.

查询上主要讲了delta prunning, scan-to-seek, resume key.

都是些小tricky, 不细讲了.



Mesa支持在线的schema修改.

主要有两种方式.

第一种方式简单粗暴, 直接复制老schema数据, 复制完成时, 利用Bigtable提供的atomic操作, 切换到新schema.

缺点是生效速度慢, 切废存储空间.

第二种方式能立即生效, 被称为linked schema change.

方法为特殊处理一些特定的schema变动, 比如添加/删除列, 然后在update的时候直接进行写新schema, query的时候对新老schema的数据进行转换和整合.

缺点是查询时可能慢点, 不过等到老的delta被完全compact到新schema的delta时就好了.
```

### Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask
```
目前先看前四章, 直接整个一起说:

传统 Volcano 模型有很大的解释开销, VectorWise 通过:
1. batch, 每次处理一批数据;
2. 按列计算, 每批处理同一种类型的数据;
使得解释的开销被均摊;

除了均摊解释开销, VectorWise 还有个好处是处理数据的 loop 内操作通常很简单;

简单的 loop, 有利于 CPU:
1. 方便 CPU 做乱序执行;
2. branch miss 后, 重做的代价较低;
这俩性质, 在论文中能得到的好处是, 使得 CPU 有更低的内存访问延迟(better latency hiding, Figure 4);

当然 VectorWise 把不同列的处理拆分成了多个 loop, 对比 compile 有两个缺点:
1. 需要的指令更多;
2. 需要物化部分中间结果;

compile 方式能把 non-blocking 的操作融合在一起(放在一个 loop 内); 

该方式优点是:
1. 会有更少的 CPU 指令;
2. 避免部分中间结果物化的开销;

缺点是 loop 内的操作可能相对复杂, 没有了 VectorWise 上面的两个好处;

上面可见 VectorWise 和 compile 方式本质上的不同在于:
1. VectorWise 倾向于把操作按列拆开, 每次处理一列, 每个 loop 内操作简单, CPU 亲和性高, 缺点是步骤增加了, 需要更多的指令, 中间结果需要物化;
2. compile 倾向于把操作融合(fuse)在一起, 每次 loop 尽量把能做的都做了, 减少不必要的指令, 也能提高 cache 命中率, 缺点是 loop 内操作相对复杂, 不方便 CPU 做优化;

compile 适合计算重的query, VectorWise 在 Join 占比大的 query 中表现比 compile 好;


文中还介绍了作者的 Vectorized Join/Agg, 不过思路比较 common, 不叙述了;

后面几张在引入 SIMD 和 并发的条件下又做了部分测试, 先不看了;

```

### CockroachDB Vectorized Execution
详细见[CRDB VEC](../slides/CRDBVectorizedExecution.pdf)

### X-Engine: An Optimized Storage Engine for Large-scale E-commerce Transaction Processing
详细见[X-Engine](../slides/X-ENGINE_ZYJ.pdf)

### Bigtable: A Distributed Storage System for Structured Data
coming soon

### What's Really New with NewSQL
coming soon

### SQL Spider
根据 TableSchema 和 DatabaseSchema 自动随机在 SQL 空间遍历，生成 SQL，用于测试；
[SQLSpider](../slides/SQLSpider.pdf)

### The Snowflake Elastic Data Warehouse
coming soon

### WiscKey: Separating Keys from Values in SSD-conscious Storage
coming soon
