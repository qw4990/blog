# BoltDB模型

接下来我们从下到上的介绍BoltDB的模型;

## 分页

#### 分页简介

BoltDB把所有的数据都维护在一个大磁盘文件上, 并分页管理它;

其格式大概如下:

```
disk: [P1|P2|P3|P4|...|Pn]
```

这些页, 可分为三种类型:

1. META\_PAGE: 存储元信息;
2. FREELIST\_PAGE: 标记某个页是否处于free状态, 用来管理所有的DATA\_PAGE;
3. DATA\_PAGE: 存储实际的数据;

下面稍微详细点的介绍这几种页;

#### **META\_PAGE**

P1和P2都是META\_PAGE, 且相互独立;

设置两个META\_PAGE的原因是用于备份, 后面会说;

下面介绍MEGA\_PAGE中一些关键字段, 并可以从中看出BoltDB的一些机制;

1. root\_pgid: BoltDB利用分页, 在磁盘维护一个B+树, 该字段表示B+树根节点所在的页号;
2. freelist\_pgid: FREELIST\_PAGE所在的页号;
3. checksum: 校验码;

#### **FREELIST\_PAGE**

就是一个数组, 记录所有了free page的页号;

FREELIST\_PAGE用来持久化这个数据;

#### **DATA\_PAGE**

持久化B+树的节点, 用来维护索引和数据;

#### 区分类型后的格式

区分类型后, 分别用MP, FP, DP来表示这些页, 则磁盘的格式大致如下:

```
disk: [MP1|MP2|DP1|DP2|...|DPi-1|FP|DPi|DPi+1|...]
```

任何页都有可能作为FREELIST PAGE使用, 在META PAGE中的freelist\_pgid记录了它当前的位置;

#### 分页的读入和缓存

由于内存有限, 一般分页后, 都需要一个PageCacher之类的组件, 来进行分页的置换;

BoltDB直接使用mmap, 直接将所有的页, 也就是整个数据大文件, 全部映射到内存内;

```
mem:  [MP1|MP2|DP1|DP2|...|DPi-1|FP|DPi|DPi+1|...]
                        ^
                        |(mmap)
                        |
disk: [MP1|MP2|DP1|DP2|...|DPi-1|FP|DPi|DPi+1|...]
```

从而省略了自己实现PageCacher的麻烦, 具体见"实现"篇;

## B+树索引

#### 节点持久化

B+树算法本身就不做过多的赘述了;

BoltDB将B+树每个节点, 都持久化到一段连续的页中;

如Node2被持久化在Page4~Page6, Node7被持久化在Page9中;

对于枝干节点, 其持久化的格式大概如下:

```
branch node: [(key1,child_pgid1)|(key2,child_pgid2)|(key3,child_pgid3)...]
```

根据B+树算法, 上述pair对根据key有序, child\_pgid指向其对应的子节点所在的页号;

对于叶子节点, 其格式大概如下:

```
leaf node: [(key1, val1)|(key2, val2)|(key3, val3)...]
```

同理, 直接存储数据内容.

-

根节点的root\_pgid存储在META\_PAGE中;

这样, 根据索引读入特定的页, 并解析成B+树节点, 能够很方便的对B+树进行遍历.

#### CopyOnWrite解决读写冲突

一般的数据库需要考虑"写写冲突", "读写冲突", 由于BoltDB只支持单写事务, 因此不存在"写写冲突";

现在考虑"读写冲突": 如果一个事务正在修改某个节点的数据, 但是还没提交, 那对于另一个读事务, 可能读到脏数据;

BoltDB使用了CopyOnWrite的方法, 对需要修改节点单保存一份;

当事务进行提交时, 将这些缓存的数据, 全部同步到磁盘;

## 图解

下面模拟一个写事务对BoltDB的影响过程;

并在更下一节进行读写冲突和事务的分析;

#### 0. 初始状态

下图中, META\_PAGE中分别有字段指向B+树的root节点和freelist page;

蓝色代表数据干净, 还未被修改;

B+树节点中的数字为其占用的页号, 如根节点的4表示根节点的内容存储在4页开始的一些页中;![](/database/boltDB/pics/bolt0.png)

#### 1. 修改/添加

某个事务调用修改/添加操作, 修改了某个叶节点的内容;

BoltDB将改叶节点的内容copy一份出来, 缓存在内存中, 如下图红色部分;![](/database/boltDB/pics/bolt1.png)

#### 2. 删除

该事务调用了删除操作, 于是影响了某个叶节点内的内容;

同理, 拷贝出一份维护在内存中;![](/database/boltDB/pics/bolt2.png)

#### 3. Rebalance-Split

在事务提交时, BoltDB会对B+树进行rebalance;

如果某个节点内容过多, B+树会对他进行分裂, 分裂后, 会影响到父节点;

假设下面的这个节点进行了分裂, 则会形成如下图:![](/database/boltDB/pics/bolt3.png)

#### 4. Rebalance-Merge

B+树的另一个性质是如果某个节点内容过少, 会和兄弟节点进行合并, 这也会影响到父节点;

假设下面这个节点进行了合并, 则形成如下图:![](/database/boltDB/pics/bolt4.png)

#### 5. 持久化简介

到目前为止, 所有的修改都维护在内存中, 下面对其进行持久化;

BoltDB的整个持久化是从下向上的;

持久化的过程中, 会为内存中维护的节点申请页用于存储, 也会释放掉被覆盖的页;

该期间会影响freelist, 其改动也是维护在内存中;

另外, 被修改的节点, 持久化时, 所使用的页号会发生改变, 所以其父节点也会受到影响, 该影响一直持续到根;

#### 6. 为B+树申请分页![](/database/boltDB/pics/bolt5.png)![](/database/boltDB/pics/bolt6.png)

因为中间两个节点的页号发生了改变, 所以其父亲节点的索引页需要更新, 于是他们的父亲节点也需要改动;

![](/database/boltDB/pics/bolt7.png)![](/database/boltDB/pics/bolt8.png)

#### 7. 为freelist申请分页

回收老freelist所使用的页, 为新freelist申请分页:![](/database/boltDB/pics/bolt10.png)

#### 8. 更新META\_PAGE

由于根节点和freelist的页号发生了改变, META PAGE也会被改变, 改变先缓存在内存中;![](/database/boltDB/pics/bolt11.png)

#### 9. 持久化B+树节点

接下来真正的持久化B+树节点;

#### ![](/database/boltDB/pics/bolt12.png)![](/database/boltDB/pics/bolt13.png)10. 持久化freelist![](/database/boltDB/pics/bolt14.png)

#### 11. 更新META PAGE到磁盘![](/database/boltDB/pics/bolt15.png)12. 最后结果![](/database/boltDB/pics/bolt16.png)

## 容错, 冲突分析及改进

#### 读写冲突分析

在事务提交前, 其改动都维护在内存中, 只对自己可见;

不会影响其他读事务;

#### 提交各阶段容错分析

接下来分析对事务的容错性;

1. 在1-8失败: 所有修改都在内存中, 对数据库文件毫无影响;
2. 在9-10失败: META\_PAGE还没有持久化, 重启后, 之前的dirty page不会对数据库目前的状态产生影响;
3. 在11失败: 使用META\_PAGE中的checksum进行校验, 如果失败, 则数据库不可用\(具体见下文Backup\);

#### Backup

Page1, Page2都被作为META\_PAGE, 其原因是可用于人工backup;

当数据库启动时, 如果meta1校验失败, 则会使用meta2;

而当数据库每次更改时, 实际上都只会修改meta1;

那么meta2什么时候会被修改呢?

BoltDB提供了一个Backup的接口, 使用这个接口后, 会将整个数据库文件, 重新拷贝一份;

并且在新文件中, 把meta1的内容写到meta2中;

.

个人觉得这个措施并没有任何作用, 而且可能会造成数据库错误;

1. 比如backup后meta2的freelist\_pgid为12;
2. 当一段时间过后, Page12内的内容可能已经被改变, 但是这段时间内, 只有meta1被改变, meta2完全不知此事;
3. 如果此时meta1被写坏, 使用meta2内的内容, 则出现了错误;

#### 一个改进点

在每次持久化Meta1之前, 先把Meta1的老数据写到Meta2的位置;

这样, 如果当写Meta2失败时, 则Meta1还是老版本, 可直接继续使用;

如果写Meta2成功后写Meta1失败, 则fallback到Meta2, 也就是事务提交之前Meta1.

