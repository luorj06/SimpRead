# SparkSQL优化

## CRO基本原理

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [sq.sf.163.com](https://sq.sf.163.com/blog/article/178698247892459520)

SQL 优化器核心执行策略主要分为两个大的方向：基于规则优化（CRO）以及基于代价优化 (CBO)

基于规则优化是一种经验式、启发式地优化思路，更多地依靠前辈总结出来的优化规则，简单易行且能够覆盖到大部分优化逻辑，但是对于核心优化算子 Join 却显得有点力不从心。

举个简单的例子，两个表执行 Join 到底应该使用 BroadcastHashJoin 还是 SortMergeJoin？当前 SparkSQL 的方式是通过手工设定参数来确定，如果一个表的数据量小于这个值就使用 BroadcastHashJoin，但是这种方案显得很不优雅，很不灵活。

基于代价优化就是为了解决这类问题，它会针对每个 Join 评估当前两张表使用每种 Join 策略的代价，根据代价估算确定一种代价最小的方案。

本文将会重点介绍基于规则的优化策略，后续文章会详细介绍基于代价的优化策略。下图中红色框框部分将是本文的介绍重点：

![](https://nos.netease.com/cloud-website-bucket/20180720174759ca63a7c5-6f92-4d69-9da2-5e5a9002881b.png)

### Tree&Rule

在介绍 SQL 优化器工作原理之前，有必要首先介绍两个重要的数据结构：Tree 和 Rule。相信无论对 SQL 优化器有无了解，都肯定知道 SQL 语法树这个概念，不错，SQL 语法树就是 SQL 语句通过编译器之后会被解析成一棵树状结构。这棵树会包含很多节点对象，每个节点都拥有特定的数据类型，同时会有 0 个或多个孩子节点（节点对象在代码中定义为 TreeNode 对象），下图是个简单的示例：

![](https://nos.netease.com/cloud-website-bucket/20180720174834e7861646-5433-45a5-9bd5-4a82793f4354.png)

如上图所示，箭头左边表达式有 3 种数据类型（Literal 表示常量、Attribute 表示变量、Add 表示动作），表示 x+(1+2)。映射到右边树状结构后，每一种数据类型就会变成一个节点。另外，Tree 还有一个非常重要的特性，可以通过一定的规则进行等价变换，如下图：

上图定义了一个等价变换规则 (Rule)：两个 Integer 类型的常量相加可以等价转换为一个 Integer 常量，这个规则其实很简单，对于上文中提到的表达式 x+(1+2) 来说就可以转变为 x+3。对于程序来讲，如何找到两个 Integer 常量呢？其实就是简单的二叉树遍历算法，每遍历到一个节点，就模式匹配当前节点为 Add、左右子节点是 Integer 常量的结构，定位到之后将此三个节点替换为一个 Literal 类型的节点。

上面用一个最简单的示例来说明等价变换规则以及如何将规则应用于语法树。在任何一个 SQL 优化器中，通常会定义大量的 Rule（后面会讲到），SQL 优化器会遍历语法树中每个节点，针对遍历到的节点模式匹配所有给定规则（Rule），如果有匹配成功的，就进行相应转换，如果所有规则都匹配失败，就继续遍历下一个节点。  

### Catalyst 工作流程

任何一个优化器工作原理都大同小异：SQL 语句首先通过 Parser 模块被解析为语法树，此棵树称为 Unresolved Logical Plan；Unresolved Logical Plan 通过 Analyzer 模块借助于数据元数据解析为 Logical Plan；此时再通过各种基于规则的优化策略进行深入优化，得到 Optimized Logical Plan；优化后的逻辑执行计划依然是逻辑的，并不能被 Spark 系统理解，此时需要将此逻辑执行计划转换为 Physical Plan；为了更好的对整个过程进行理解，下文通过一个简单示例进行解释。  

**Parser**

Parser 简单来说是将 SQL 字符串切分成一个一个 Token，再根据一定语义规则解析为一棵语法树。Parser 模块目前基本都使用第三方类库 ANTLR 进行实现，比如 Hive、 Presto、SparkSQL 等。下图是一个示例性的 SQL 语句（有两张表，其中 people 表主要存储用户基本信息，score 表存储用户的各种成绩），通过 Parser 解析后的 AST 语法树如右图所示：

![](https://nos.netease.com/cloud-website-bucket/20180720174904e7797de3-8cd8-46f1-8f25-127194456927.png)

**Analyzer**

通过解析后的逻辑执行计划基本有了骨架，但是系统并不知道 score、sum 这些都是些什么鬼，此时需要基本的元数据信息来表达这些词素，最重要的元数据信息主要包括两部分：表的 Scheme 和基本函数信息，表的 scheme 主要包括表的基本定义（列名、数据类型）、表的数据格式（Json、Text）、表的物理位置等，基本函数信息主要指类信息。

Analyzer 会再次遍历整个语法树，对树上的每个节点进行数据类型绑定以及函数绑定，比如 people 词素会根据元数据表信息解析为包含 age、id 以及 name 三列的表，people.age 会被解析为数据类型为 int 的变量，sum 会被解析为特定的聚合函数，如下图所示：

![](https://nos.netease.com/cloud-website-bucket/201807201749187ade4bf5-4ed9-4de2-97cb-49b20ce4f032.png)

SparkSQL 中 Analyzer 定义了各种解析规则，有兴趣深入了解的童鞋可以查看 Analyzer 类，其中定义了基本的解析规则，如下：

![](https://nos.netease.com/cloud-website-bucket/201807201749336c0403bd-2b00-45b7-8146-7500e46ed514.png)

**Optimizer**

优化器是整个 Catalyst 的核心，上文提到优化器分为基于规则优化和基于代价优化两种，当前 SparkSQL 2.1 依然没有很好的支持基于代价优化（下文细讲），此处只介绍基于规则的优化策略，基于规则的优化策略实际上就是对语法树进行一次遍历，模式匹配能够满足特定规则的节点，再进行相应的等价转换。因此，基于规则优化说到底就是一棵树等价地转换为另一棵树。SQL 中经典的优化规则有很多，下文结合示例介绍三种比较常见的规则：谓词下推（Predicate Pushdown）、常量累加（Constant Folding）和列值裁剪（Column Pruning）。

![](https://nos.netease.com/cloud-website-bucket/201807201749460db6682e-382e-456a-98ce-6aeab7ffa9fb.png)

上图左边是经过 Analyzer 解析后的语法树，语法树中两个表先做 join，之后再使用 age>10 对结果进行过滤。大家知道 join 算子通常是一个非常耗时的算子，耗时多少一般取决于参与 join 的两个表的大小，如果能够减少参与 join 两表的大小，就可以大大降低 join 算子所需时间。谓词下推就是这样一种功能，它会将过滤操作下推到 join 之前进行，上图中过滤条件 age>0 以及 id!=null 两个条件就分别下推到了 join 之前。这样，系统在扫描数据的时候就对数据进行了过滤，参与 join 的数据量将会得到显著的减少，join 耗时必然也会降低。

![](https://nos.netease.com/cloud-website-bucket/201807201749595713b3eb-0371-4d06-8244-97aa1d672f18.png)

常量累加其实很简单，就是上文中提到的规则  x+(1+2)  -> x+3，虽然是一个很小的改动，但是意义巨大。示例如果没有进行优化的话，每一条结果都需要执行一次 100+80 的操作，然后再与变量 math_score 以及 english_score 相加，而优化后就不需要再执行 100+80 操作。

![](https://nos.netease.com/cloud-website-bucket/2018072017501287aae300-0427-4d37-b9ff-159f938a42e8.png)

列值裁剪是另一个经典的规则，示例中对于 people 表来说，并不需要扫描它的所有列值，而只需要列值 id，所以在扫描 people 之后需要将其他列进行裁剪，只留下列 id。这个优化一方面大幅度减少了网络、内存数据量消耗，另一方面对于列存数据库（Parquet）来说大大提高了扫描效率。

除此之外，Catalyst 还定义了很多其他优化规则，有兴趣深入了解的童鞋可以查看 Optimizer 类，下图简单的截取一部分规则：

![](https://nos.netease.com/cloud-website-bucket/20180720175026c9441b57-28de-4352-a916-f289e1ecbb84.png)

至此，逻辑执行计划已经得到了比较完善的优化，然而，逻辑执行计划依然没办法真正执行，他们只是逻辑上可行，实际上 Spark 并不知道如何去执行这个东西。比如 Join 只是一个抽象概念，代表两个表根据相同的 id 进行合并，然而具体怎么实现这个合并，逻辑执行计划并没有说明。

![](https://nos.netease.com/cloud-website-bucket/20180720175040ab37bd80-d96e-4e0c-bf16-12e9c82197a0.png)

此时就需要将逻辑执行计划转换为物理执行计划，将逻辑上可行的执行计划变为 Spark 可以真正执行的计划。比如 Join 算子，Spark 根据不同场景为该算子制定了不同的算法策略，有 BroadcastHashJoin、ShuffleHashJoin 以及 SortMergeJoin 等（可以将 Join 理解为一个接口，BroadcastHashJoin 是其中一个具体实现），物理执行计划实际上就是在这些具体实现中挑选一个耗时最小的算法实现，这个过程涉及到基于代价优化策略，后续文章细讲。  

### SparkSQL 执行计划

至此，笔者通过一个简单的示例完整的介绍了 Catalyst 的整个工作流程，包括 Parser 阶段、Analyzer 阶段、Optimize 阶段以及 Physical Planning 阶段。有同学可能会比较感兴趣 Spark 环境下如何查看一条具体的 SQL 的整个过程，在此介绍两种方法：

1. 使用 queryExecution 方法查看逻辑执行计划，使用 explain 方法查看物理执行计划，分别如下所示：

![](https://nos.netease.com/cloud-website-bucket/20180720175058b322bb28-5db6-4205-b3f8-e12e505f966a.png)

![](https://nos.netease.com/cloud-website-bucket/20180720175115abd28964-ba95-4f0b-aba7-776615e8e04e.png)

## CBO 基本原理

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [sq.sf.163.com](https://sq.sf.163.com/blog/article/178255009191530496)

CRO 是一种经验式、启发式的优化思路，优化规则都已经预先定义好，只需要将 SQL 往这些规则上套就可以。 说白了，CRO 就像是一个经验丰富的老司机，基本套路全都知道 。

然而世界上有一种东西叫做 - 不按套路来，与其说它不按套路来，倒不如说它本身就没有什么套路而言。最典型的莫过于复杂 Join 算子，对于这些复杂的 Join 来说，通常有两个对优化相当重要的问题需要决定：  

1. 该 Join 应该选择哪种策略来执行？BroadcastJoin or ShuffleHashJoin or SortMergeJoin？不同的执行策略对系统的资源要求不同，执行效率也有天壤之别，同一个 SQL，选择到合适的策略执行可能只需要几秒钟，而如果没有选择到合适的执行策略就可能会导致系统 OOM。

2. 对于雪花模型或者星型模型来讲，多表 Join 应该选择什么样的顺序执行？不同的 Join 顺序意味着不同的执行效率，比如 A join B join C，A、B 表都很大，C 表很小，那 A join B 很显然需要大量的系统资源来运算，执行时间肯定不会短。而如果使用 A join C join B 的执行顺序，因为 C 表很小，所以 A join C 会很快得到结果，而且结果集会很小，再使用小的结果集 join B，结果必然也会很小。

首先来看第一个问题，当前 SparkSQL 会让用户指定参数'spark.sql.autoBroadcastJoinThreshold’来决定是否采用 BroadcastJoin 策略，简单来说，它会选择参与 Join 的两表中的小表大小与该值进行对比，如果小表大小小于该配置值，就将此表进行广播；否则采用 SortMergeJoin 策略。对于 SparkSQL 采取的方式，有两个问题需要深入分析：

*   参数'spark.sql.autoBroadcastJoinThreshold’指定的是表的大小（size），而不是条数。这样完全合理吗？我们知道 Join 算子会将两表中具有相同 key 的记录合并在一起，因此 Join 的复杂度只与两表的数据条数有关，而与表大小（size）没有直接的关系。这样一想，其实参数'spark.sql.autoBroadcastJoinThreshold’应该更多地是考虑广播的代价，而不是 Join 本身的代价。  

*   之前 Catalyst 文章中我们讲到谓词下推规则，Catalyst 会将很多过滤条件下推到 Join 之前，因此参与 Join 的两表大小并不应该是两张原始表的大小，而是经过过滤后的表数据大小。因此，单纯的知道原始表大小还远远不够，Join 优化还需要评估过滤后表数据大小以及表数据条数。

对于第二个问题，也面临和第一个问题同样的两点缺陷，举个简单的例子：

*   SQL：select * from A , B , C where A.id = B.b_id and A.id = C.c_id and C.c_id > 100  

*   假设：A、B、C 总纪录大小分别是 100m，40m，60m，C.c_id > 100 过滤后 C 的总纪录数会降到 10m

上述 SQL 是一个典型的 A join B join C 的多 Join 示例，很显然，A 肯定在最前面，现在问题是 B 和 C 的 Join 顺序，是 A join B join C 还是 A join C join B。对于上面的示例，优化器会有两种最基本的选择，第一就是按照用户手写的 Join 顺序执行，即按照‘A.id = B.b_id and A.id = C.c_id ’顺序， 得到的执行顺序是 A join B join C。第二是按照 A、B 、C 三表的原始大小进行组织排序，原始表小的先 Join，原始表大的后 Join，适用这种规则得到的顺序依然是 A join B join C，因为 B 的记录大小小于 C 的记录大小。

同样的道理，第一个缺陷很明显，记录大小并不能精确作为 Join 代价的计算依据，而应该是记录条数。第二就是对于过滤条件的忽略，上述示例中 C 经过过滤后的大小降到 10m，明显小于 B 的 40m，因此实际上应该执行的顺序为 A join C join B，与上述得到的结果刚好相反。

可见，基于规则的优化策略并不适合复杂 Join 的优化，此时就需要另一种优化策略 - 基于代价优化（CBO）。基于代价优化策略实际只做两件事，就是解决上文中提到的两个问题：

1.  解决参与 Join 的数据表大小并不能完全作为计算 Join 代价依据的问题，而应该加入数据记录条数这个维度  

2.  解决 Join 代价计算应该考虑谓词下推（等）条件的影响，不能仅仅关注原始表的大小。这个问题看似简单，实际上很复杂，需要优化器将逻辑执行计划树上的每一个节点的 <数据量，数据条数> 都评估出来，这样的话，评估 Join 的执行策略、执行顺序就不再以原始表大小作为依据，而是真实参与 Join 的两个数据集的大小条数作为依据。

### CBO 实现思路

经过上文的解释，可以明确 CBO 的本质就是计算 LogionPlan 每个节点的输出数据大小与数据条数，作为复杂 Join 算子的代价计算依据。逻辑执行计划树的叶子节点是原始数据，往上会经过各种过滤条件以及其他函数的限制，父节点依赖于子节点。整个过程可以变换为子节点经过特定计算评估父节点的输出，计算出来之后父节点将会作为上一层的子节点往上计算。因此，CBO 可以分解为两步：

1.  一次性计算出原始数据的相关数据  

2.  再对每类节点制定一种对应的评估规则就可以自下往上评估出所有节点的代价值

**一次性计算出原始表的相关数据**  

这个操作是 CBO 最基础的一项工作，在计算之前，我们需要明确 “相关数据” 是什么？这里给出核心的统计信息如下：

*   estimatedSize: 每个 LogicalPlan 节点输出数据大小（解压）  

*   rowCount: 每个 LogicalPlan 节点输出数据总条数  

*   basicStats: 基本列信息，包括列类型、Max、Min、number of nulls, number of distinct values, max column length, average column length 等  

*   Histograms: Histograms of columns, i.e., equi-width histogram (for numeric and string types) and equi-height histogram (only for numeric types).

至于为什么要统计这么多数据，下文会讲到。现在再来看如何进行统计，有两种比较可行的方案：

1. 打开所有表扫描一遍，这样最简单，而且统计信息准确，缺点是对于大表来说代价比较大。hive 和 impala 目前都采用的这种方式：

（1）hive 统计原始表命令：analyse table ***

（2）impala 统计原始表明了：compute stats ***

2. 针对一些大表，扫描一遍代价太大，可以采用采样（sample）的方式统计计算

**代价评估规则 & 计算所有节点统计信息**

代价评估规则意思是说在当前子节点统计信息的基础上，计算父节点相关统计信息的一套规则。 对于不同谓词节点，评估规则必然不一样，比如 fliter、group by、limit 等等的评估规则不同。 假如现在已经知道表 C 的基本统计信息，对于 SQL： select * from A , B , C where A.id = B.b_id and A.id = C.c_id and C.c_id > N  这个条件，如何评估经过 C.c_id > N 过滤后的基本统计信息。我们来看看：

1. 假设当前已知 C 列的最小值 c_id.Min、最大值 c_id.Max 以及总行数 c_id.Distinct，如下图所示：

![](https://nos.netease.com/cloud-website-bucket/201807191229252e762b81-44a0-4ea6-854c-71da03816b55.png) 

  

2. 现在分别有三种情况需要说明，其一是 N 小于 c_id.Min，其二是 N 大于 c_id.Max，其三是 N 介于 c_id.Min 和 c_id.Max 之间。前两种场景是第三种场景的特殊情况，这里简单的针对第三种场景说明。如下图所示：

![](https://nos.netease.com/cloud-website-bucket/20180719122931bd9ed1ea-c54b-444c-b1a7-6f5c78d350bc.png) 

  

在 C.c_id > N 过滤条件下，c_id.Min 会增大到 N，c_id.Max 保持不变。而过滤后总行数 c_id.distinct(after filter) ＝ (c_id.Max - N) / (c_id.Max - c_id.Min) * c_id.distinct(before filter)

当然，上述计算只是示意性计算，真实算法会复杂很多。另外，如果大家对 group by 、limit 等谓词的评估规则比较感兴趣的话，可以阅读[SparkSQL CBO 设计文档](https://issues.apache.org/jira/secure/attachment/12823839/Spark_CBO_Design_Spec.pdf)

至此，通过各种评估规则就可以计算出语法树中所有节点的基本统计信息，当然最重要的是参与 Join 的数据集节点的统计信息。最后只需要根据这些统计信息选择最优的 Join 算法以及 Join 顺序，最终得到最优的物理执行计划。

**Hive - CBO 优化效果**

Hive 本身没有去从头实现一个 SQL 优化器，而是借助于 [Apache Calcite](http://calcite.apache.org/) ，Calcite 是一个开源的、基于 CBO 的企业级 SQL 查询优化框架，目前包括 Hive、Phoniex、Kylin 以及 Flink 等项目都使用了 Calcite 作为其执行优化器，这也很好理解，执行优化器本来就可以抽象成一个系统模块，并没有必要花费大量时间去重复造轮子。

hortonworks 曾经对 Hive 的 CBO 特性做了相关的测试，测试结果认为 CBO 至少对查询有三个重要的影响：Join ordering optimization、Bushy join support 以及 Join simplification，本文只简单介绍一下 Join ordering optimization，有兴趣的同学可以继续阅读[这篇文章](http://hortonworks.com/blog/hive-0-14-cost-based-optimizer-cbo-technical-overview/)来更多地了解其他两个重要影响。（下面数据以及示意图也来自于该篇文章，特此注明）

hortonworks 对 TPCDS 的部分 Query 进行了研究，发现对于大部分星型 \ 雪花模型，都存在多 Join 问题，这些 Join 顺序如果组织不好，性能就会很差，如果组织得当，性能就会很好。比如 Query Q3：  

```
select
    dt.d_year,
    item.i_brand_id brand_id,
    item.i_brand brand,
    sum(ss_ext_sales_price) sum_agg
from
    date_dim dt,
    store_sales,
    item
where
    dt.d_date_sk = store_sales.ss_sold_date_sk
and store_sales.ss_item_sk = item.i_item_sk
and item.i_manufact_id =436
and dt.d_moy =12
groupby dt.d_year , item.i_brand , item.i_brand_id
order by dt.d_year , sum_agg desc , brand_id
limit 10
```

上述 Query 涉及到 3 张表，一张事实表 store_sales（数据量大）和两张维度表（数据量小），三表之间的关系如下图所示：

这里就涉及上文提到的 Join 顺序问题，从原始表来看，date_dime 有 73049 条记录，而 item 有 462000 条记录。很显然，如果没有其他暗示的话，Join 顺序必然是 store_sales join date_time join item。但是，where 条件中还带有两个条件，CBO 会根据过滤条件对过滤后的数据进行评估，结果如下：

<table><tbody><tr><td><p>Table</p></td><td><p>Cardinality</p></td><td><p>Cardinality after filter&nbsp;</p></td><td><p>Selectivity</p></td></tr><tr><td><p>date_dim</p></td><td><p>73,049</p></td><td><p>6200</p></td><td><p>8.5%</p></td></tr><tr><td><p>item</p></td><td><p>462,000</p></td><td><p>484</p></td><td><p>0.1%</p></td></tr></tbody></table>

根据上表所示，过滤后的数据量 item 明显比 date_dim 小的多，剧情反转的有点快。于是乎，经过 CBO 之后 Join 顺序就变成了 store_sales join item join date_time，为了进一步确认，可以在开启 CBO 前后分别记录该 SQL 的执行计划，如下图所示：

![](https://nos.netease.com/cloud-website-bucket/20180719122957e682bf1c-488c-4e7d-8125-bb8817dc294b.png) 

  

左图是未开启 CBO 特性时 Q3 的执行计划，store_sales 先与 date_dim 进行 join，join 后的中间结果数据集有 140 亿条。而再看右图，store_sales 先于 item 进行 join，中间结果只有 8200w 条。很显然，后者执行效率会更高，实践出真知，来看看两者的实际执行时间：

<table><tbody><tr><td>Test</td><td>Query Response Time(seconds)</td><td>Intermediate Rows</td><td>CPU(seconds)</td></tr><tr><td>Q3 CBO OFF</td><td>255</td><td>13,987,506,884</td><td>51,967</td></tr><tr><td>Q3 CBO ON</td><td>142</td><td>86,217,653</td><td>35,036</td></tr></tbody></table>

上图很明显的看出 Q3 在 CBO 的优化下性能将近提升了 1 倍，与此同时，CPU 资源使用率也降低了一半左右。不得不说，TPCDS 中有很多相似的 Query，有兴趣的同学可以深入进一步深入了解。

**Impala - CBO 优化效果**

和 Hive 优化的原理相同，也是针对复杂 join 的执行顺序、Join 的执行策略选择优化等方面进行的优化，本人使用 TPC-DS 对 Impala 在开启 CBO 特性前后的部分 Query 进行了性能测试，测试结果如下图所示：  

![](https://nos.netease.com/cloud-website-bucket/201807191230068ce8d020-5cf7-42d8-8db6-b56ded82df38.png) 

  