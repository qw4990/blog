## 论文简结

### Volcano-An Extensible and Parallel Query Evaluation System
看得比较快, 主要引入了Open-Next-Close的volcano模型.


### Access Path Selection in a Relational Database Management System
看了前4节, 主要引入了AcessPath和Cost-Based, 在多条AcessPath上计算代价, 选取最优解.
在计算代价的部分, 利用简单的统计信息, 提出了一些简单的算法.
不过代价计算算法和其存储引擎比较绑定, 且对数据假设太过简单.


### MonetDB/X100: Hyper-Pipelining Query Execution
Section 2简述CPU工作原理, 点出对性能影响比较大的几个因素:
1. cpu cache
2. branch predict
3. loop pipeline
其中提到了几个比较有意思的数据:
1. 30%的指令为内存读写
2. 内存读写需要几十ns, 需要CPU等待几百个cycle

Section 3分析了TPC-H Q1在Mysql和MIL运行情况
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


### Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age
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


### The Volcano Optimizer Generator: Extensibility and Efficient Search
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


### Orca: A Modular Query Optimizer Architecture for Big Data
Section 3 讲整体结构, 由于Orca是独立于Database的单独的优化器, 因此定义了一个DXL语言来交换信息.
内部结构方面, 较为重要的是memo这个结构, 源自cascade framework.
1. 每个query都会对应一个memo, memo表示这个query的整个执行计划搜寻空间.
2. memo包含多个group, 多个group之间有依赖关系, 每个group表示一个算子的子搜寻空间, 如InnerJoin.
3. group包含多个group expression, 表示多种实现算子的方法, 如InnerJoin可能有InnerNLJoin/InnerHashJoin等多种实现方式.

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


### Efficiently Compiling Efficient Query Plans for Modern Hardware
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


### Large-scale Incremental Processing Using Distributed Transactions and Notifications
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
