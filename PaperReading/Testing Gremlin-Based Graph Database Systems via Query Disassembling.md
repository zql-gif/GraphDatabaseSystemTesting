## links
* 论文链接：[Testing Gremlin-Based Graph Database Systems via Query Disassembling | Proceedings of the 33rd ACM SIGSOFT International Symposium on Software Testing and Analysis](https://dl.acm.org/doi/10.1145/3650212.3680392#:~:text=In%20this%20paper%2C%20we%20propose%20Query%20Di%20sassembling,to%20automatically%20detect%20logic%20bugs%20in%20Gremlin-based%20GDBs.)
* PPT链接：[幻灯片 1](https://is.cas.cn/ztzl2016/2024xsnh/2024hbzs/202408/P020240828552102655927.pdf)

## Authors
Institute of Software at CAS, China(中科院)
Yingying Zheng∗, Wensheng Dou∗†‡
## Abstract
Graph Database Systems (GDBs) support effciently storing and retrieving graph data, and have become a critical component in many important applications. **Many widely-used GDBs utilize the Gremlin query language** to create, modify, and retrieve data in graph databases, in which developers can assemble a sequence of Gremlin APIs to perform a complex query. However, incorrect implementations and optimizations of GDBs can introduce **logic
bugs**, which can cause Gremlin queries to return incorrect query
results, e.g., omitting vertices in a graph database.

In this paper, we propose **Query Disassembling (QuDi), an effective testing technique to automatically detect logic bugs in Gremlin-based GDBs.** Given a Gremlin query Q, QuDi disassembles Q into a sequence of atomic graph traversals TList, which shares the equivalent execution semantics with Q. If the execution results of Q and TList are different, a logic bug is revealed in the target GDB. We evaluate QuDi on six popular GDBs, and have found 25 logic bugs in these GDBs, 10 of which have been confirmed as previously-unknown bugs by GDB developers.

Keywords: Graph database systems, graph traversal, logic bug, bug detection

背景和问题：图数据库系统（Graph Database Systems, GDBs）能够高效地存储和检索图数据，并已成为许多重要应用中的关键组件。许多广泛使用的GDBs采用Gremlin查询语言来创建、修改和检索图数据库中的数据，开发者可以通过组合一系列Gremlin API来执行复杂的查询。然而，GDBs的错误实现和优化可能会引入逻辑错误，导致Gremlin查询返回错误的结果，例如遗漏图数据库中的某些顶点。

方法：在本文中，**我们提出了一种有效的测试技术——查询分解（Query Disassembling, QuDi），用于自动检测基于Gremlin的GDBs中的逻辑错误。给定一个Gremlin查询Q，QuDi将Q分解为一系列原子图遍历操作TList，这些操作与Q具有等效的执行语义。如果Q和TList的执行结果不同，则表明目标GDB中存在逻辑错误。** 我们在六个流行的GDBs上评估了QuDi，并发现了25个逻辑错误，其中10个已被GDB开发者确认为此前未知的错误。

## Introduction

**图数据库系统（Graph Database Systems, GDBs）**
* GDBs定义：支持高效的图数据存储和查询，图数据由顶点和边组成。
* GDBs重要应用场景：社交网络[35]、知识图谱[26, 45]和欺诈检测[50]。
* GDBs应用广泛：最新的DB-Engines图数据库排名[3]显示目前已有41个广泛使用的GDBs。

**Gremlin: procedural Gremlin query language**
* declarative Structured Query Language: relational database systems使用声明式的结构化查询语言访问关系数据。 
* procedural query language: **GDBs没有标准化的方式来访问图数据，通常使用自己的查询语言**，例如NebulaGraph中的nGQL[11]和TigerGraph中的GSQL[32]。
* Gremlin-based GDBs: 在DB-Engines排名[3]中，约有一半的GDBs支持Gremlin查询语言。
* Gremlin特点:  Gremlin查询语言提供了一组Gremlin API，用于创建、修改和检索图数据。


为了提高Gremlin查询的性能，图数据库系统（GDBs）采用复杂的执行和优化策略（例如重新排序过滤操作以优先执行成本较低的操作，以及合并过滤条件以实现高效的图查询）。
然而，这种复杂性无疑为GDBs的正确性带来了重大挑战。
GDBs的错误实现和优化可能会引入逻辑错误。


本文工具检测到的示例逻辑错误：ArcadeDB#500， Gremlin
* 查询语句解释：g.V()检索fig2 中所有顶点->保留标签为person且age属性小于30的顶点->保留有标签person或者book的顶点
* 正确查询结果：{1,4}
* 返回查询结果(错误)：{1, 2, 3, 4}
* ArcadeDB开发者的解释：这个逻辑错误是由于多个过滤条件组合的实现错误导致的，并迅速修复了它。（详细解释：问题的原因出在优化层，该层通过拦截索引上的查询并利用索引来加速性能。但在这种情况下，由于存在多个条件，无法单独使用索引。）
* bug链接：[Gremlin api "hasLabel" after "has" returns unexpected result · Issue #500 · ArcadeData/arcadedb](https://github.com/ArcadeData/arcadedb/issues/500)


![LogicBugInArcadeDB#500](../Pictures/LogicBugInArcadeDB_500.png)

![LogicBugInArcadeDB#500_graph](../Pictures/LogicBugInArcadeDB_500_graph.png)


> [!NOTE]  ArcadeDB#500解释
> 这条查询：
g.V().has('vl1', 'vp1', lt(2)).hasLabel('vl1', 'vl2', 'vl3').count()
涉及了多个条件，这会导致优化器无法有效地利用索引，从而产生逻辑错误。具体分析如下：
1.查询分解：
`has('vl1', 'vp1', lt(2))`：这表示查找属性 `vl1` 中名为 `vp1` 且值小于 2 的顶点。
 `hasLabel('vl1', 'vl2', 'vl3')`：这个条件要求顶点的标签是 `vl1`、`vl2` 或 `vl3` 中的一个。
2.逻辑错误的原因：
**索引优化的限制**：一般情况下，数据库的查询优化器会尝试利用索引来加速查询。如果存在索引可以加速对 `vl1` 和 `vp1` 属性的查询（例如，基于 `vl1` 和 `vp1` 的联合索引），优化器会尝试使用该索引来快速筛选符合条件的顶点。然而，当查询中有多个条件时，特别是涉及 `hasLabel` 和属性条件的组合时，索引的应用可能变得复杂或无法同时覆盖所有条件。
**多个条件的冲突**：在这个查询中，`hasLabel('vl1', 'vl2', 'vl3')` 需要在查询中应用三个标签条件，而 `has('vl1', 'vp1', lt(2))` 则是一个属性条件，要求 `vl1` 的 `vp1` 属性小于 2。这两部分条件可能会导致优化器无法有效地将索引同时应用到属性查询和标签查询的结合上。
标签 (`hasLabel`) 通常是基于顶点的标签类型进行筛选，而属性 (`has`) 查询是基于顶点的属性进行筛选。由于这两个条件的过滤方式不同，优化器可能无法找到一种高效的方式来同时使用索引，从而导致查询性能下降或者不能使用索引来加速查询。
**结论**：该查询的逻辑错误源于查询条件中涉及多个不同类型的过滤（属性和标签），导致索引优化器无法同时有效地使用索引。需要通过优化查询逻辑或调整索引设计来避免这个问题。
