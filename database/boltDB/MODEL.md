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
2. FREELIST\_PAGE: 存储pagefreelist, pagefreelist用来管理所有的DATA\_PAGE;
3. DATA\_PAGE: 存储实际的数据;

下面稍微详细点的介绍这几种页;



#### **META\_PAGE**

P1和P2都是META\_PAGE, 且相互独立;

设置两个META\_PAGE的原因是用于备份, 后面会将;

下面介绍MEGA\_PAGE中一些关键字段, 并可以从中看出BoltDB的一些机制;

1. root\_pgid: BoltDB利用分页, 在磁盘维护一个B+树, 该字段表示B+树根节点所在的页号;
2. freelist\_pgid: FREELIST\_PAGE所在的页号;
3. checksum: 校验码;

#### **FREELIST\_PAGE**

就是一个数组, 记录所有了free page的页号;

#### **DATA\_PAGE**

用于存储B+树的节点, 用来维护索引和数据;



#### 区分类型后的格式

区分类型后, 分别用MP, FP, DP来表示这些页, 则磁盘的格式大致如下:

```
disk: [MP1|MP2|DP1|DP2|...|DPi-1|FP|DPi|DPi+1|...]
```

任何页都有可能作为FP使用, 在MP中记录了FP当前的位置;



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

从而省略了自己实现PageCacher的麻烦;

在之后的所有描述中, 除非特别说明, 否则所指的页, 都指的是内存中的页;





