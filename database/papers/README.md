## 论文简结

### Volcano-An Extensible and Parallel Query Evaluation System
看得比较快, 主要引入了Open-Next-Close的volcano模型.

### Access Path Selection in a Relational Database Management System
看了前4节, 主要引入了AcessPath和Cost-Based, 在多条AcessPath上计算代价, 选取最优解.
在计算代价的部分, 利用简单的统计信息, 提出了一些简单的算法.
不过代价计算算法和其存储引擎比较绑定, 且对数据假设太过简单.

### MonetDB/X100: Hyper-Pipelining Query Execution
看得比较细.

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
