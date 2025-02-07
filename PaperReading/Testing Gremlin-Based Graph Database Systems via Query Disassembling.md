## links
* 论文链接：[Testing Gremlin-Based Graph Database Systems via Query Disassembling | Proceedings of the 33rd ACM SIGSOFT International Symposium on Software Testing and Analysis](https://dl.acm.org/doi/10.1145/3650212.3680392#:~:text=In%20this%20paper%2C%20we%20propose%20Query%20Di%20sassembling,to%20automatically%20detect%20logic%20bugs%20in%20Gremlin-based%20GDBs.)
* PPT链接：[幻灯片 1](https://is.cas.cn/ztzl2016/2024xsnh/2024hbzs/202408/P020240828552102655927.pdf)

## Authors
Institute of Software at CAS, China(中科院)
Yingying Zheng∗, Wensheng Dou∗†‡
## 0 Abstract
Graph Database Systems (GDBs) support effciently storing and retrieving graph data, and have become a critical component in many important applications. **Many widely-used GDBs utilize the Gremlin query language** to create, modify, and retrieve data in graph databases, in which developers can assemble a sequence of Gremlin APIs to perform a complex query. However, incorrect implementations and optimizations of GDBs can introduce **logic
bugs**, which can cause Gremlin queries to return incorrect query
results, e.g., omitting vertices in a graph database.

In this paper, we propose **Query Disassembling (QuDi), an effective testing technique to automatically detect logic bugs in Gremlin-based GDBs.** Given a Gremlin query Q, QuDi disassembles Q into a sequence of atomic graph traversals TList, which shares the equivalent execution semantics with Q. If the execution results of Q and TList are different, a logic bug is revealed in the target GDB. We evaluate QuDi on six popular GDBs, and have found 25 logic bugs in these GDBs, 10 of which have been confirmed as previously-unknown bugs by GDB developers.

Keywords: Graph database systems, graph traversal, logic bug, bug detection

背景和问题：图数据库系统（Graph Database Systems, GDBs）能够高效地存储和检索图数据，并已成为许多重要应用中的关键组件。许多广泛使用的GDBs采用Gremlin查询语言来创建、修改和检索图数据库中的数据，开发者可以通过组合一系列Gremlin API来执行复杂的查询。然而，GDBs的错误实现和优化可能会引入逻辑错误，导致Gremlin查询返回错误的结果，例如遗漏图数据库中的某些顶点。

方法：在本文中，**我们提出了一种有效的测试技术——查询分解（Query Disassembling, QuDi），用于自动检测基于Gremlin的GDBs中的逻辑错误。给定一个Gremlin查询Q，QuDi将Q分解为一系列原子图遍历操作TList，这些操作与Q具有等效的执行语义。如果Q和TList的执行结果不同，则表明目标GDB中存在逻辑错误。** 我们在六个流行的GDBs上评估了QuDi，并发现了25个逻辑错误，其中10个已被GDB开发者确认为此前未知的错误。

## 1 Introduction

### 1.1 图数据库系统（Graph Database Systems, GDBs）
* GDBs定义：支持高效的图数据存储和查询，图数据由顶点和边组成。
* GDBs重要应用场景：社交网络[35]、知识图谱[26, 45]和欺诈检测[50]。
* GDBs应用广泛：最新的DB-Engines图数据库排名[3]显示目前已有41个广泛使用的GDBs。

### 1.2 Gremlin: procedural Gremlin query language
* declarative Structured Query Language: relational database systems使用声明式的结构化查询语言访问关系数据。 
* procedural query language: **GDBs没有标准化的方式来访问图数据，通常使用自己的查询语言**，例如NebulaGraph中的nGQL[11]和TigerGraph中的GSQL[32]。
* Gremlin-based GDBs: 在DB-Engines排名[3]中，约有一半的GDBs支持Gremlin查询语言。
* Gremlin特点:  Gremlin查询语言提供了一组Gremlin API，用于创建、修改和检索图数据。
* **Gremlin中逻辑错误的诱因：为提高查询的性能，GDBs采用复杂的执行和优化策略（例如重新排序过滤操作以优先执行成本较低的操作，以及合并过滤条件以实现高效的图查询）。这种GDBs的错误实现和优化可能会引入逻辑错误。**

### 1.3 本文工具检测到的示例逻辑错误：ArcadeDB#500， Gremlin
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

### 1.4 现有工作及局限性

1. GDB的逻辑错误检测的挑战
* 缺乏一种有效的测试预言来自动测试并验证GDB在给定Gremlin查询下的行为是否正确。
* 针对关系型数据库系统的测试预言[27, 30, 37, 51–53, 56, 57]，无法直接应用于基于Gremlin的GDBs。
2. **差异测试（Differential Testing, DT）**
* 定义：将相同查询输入到多个 GDBs 中，旨在发现它们之间的不一致结果。
* **GDsmith [北大]**：首次使用差异测试检测基于Cypher的GDB
	* GDsmith 采用了**骨架生成和补全技术**确保随机生成的 Cypher 查询满足语义要求，利用**三种结构变异策略**增加返回非空结果的 Cypher 查询的概率，并**根据属性键的先前频率动态选择属性键以提高生成的 Cypher 查询的行为多样性**。GDsmith 成功地在三个流行的开源图数据库引擎的发布版本上检测到 27 个先前未知的漏洞。
* **Grand [中科院技术研究所]**：首次使用差异测试检测基于Gremlin的GDB
	* Grand，一种用于自动发现采用Gremlin检索图数据的图数据库系统中逻辑错误的随机差异测试方法，并在六种广泛使用的图数据库系统中发现了21个先前未知的逻辑错误。
	- 技术路线:为多个GDBs构建语义等价数据库；基于模型的查询生成方法来生成可能返回非空结果的有效Gremlin查询；使用数据映射方法统一不同GDBs的查询结果格式。
![GrandOverview](../Pictures/GrandOverview.png)
* RD [62], [幻灯片 1](https://is.cas.cn/ztzl2016/2023xsnh/2023hbzs/202309/P020231228505999042717.pdf)
* 局限性：**所有 GDBs 返回相同的结果并不一定意味着正确性。** 使用不一致性作为差异测试的测试预言（oracle）会带来两个问题：  
	* 被测GDBs 可能存在相同逻辑错误：它无法检测所有被测试 GDBs 中都存在的错误，因为这些 GDBs 可能始终返回相同的不正确结果  
	* 被测功能有限：只能测试被测 GDBs 之间重叠的通用功能，而无法测试特定 GDB 的独特功能

3. **蜕变测试（Metamorphic Testing, MT）**
* 定义：精心设计蜕变关系（Metamorphic Relations，MRs），对输入进行变异，检查输出是否违反所规定的属性来检测错误。蜕变关系缓解了差异测试预言的问题，不需要存在多个类似的程序实现
* GDBMeter[34] : 
	* GDBMeter 是唯一采用蜕变测试（Metamorphic Testing，MT）的图形数据库测试工具。它将关系数据库中使用的三元查询分区（ternary query partitioning，TLP ）[42] 的思想移植到 GDB 中。它将给定的查询拆分为三个派生子查询，其中谓词分别被评估为 `TRUE`、`FALSE` 和 `IS NULL`。然后，它验证派生结果集的并集与原始结果集之间的一致性。
* 局限性：**现有的蜕变测试（MT）解决方案上述方法都是为使用SQL类或其他图查询语言的数据库系统设计的，并未关注 GDBs 中的图原生结构，因此未能提出一套图感知的MR，这限制了它们的有效性**。  
	* 覆盖语法有限（局限一）：GDBs 的图查询语言包含了与图原生结构相关的丰富语法，而现有的 MT 方法仅覆盖了有限的语法。例如，常用的路径遍历（path traversal）语法和 `𝑢𝑛𝑖𝑜𝑛` 子句在之前的工作中并未得到支持。  
	* 缺乏针对图形原生结构的有效测试预言（局限二）：不能充分识别与图形原生结构相关的逻辑错误。GDBMeter 直接复用关系数据库系统中的三元查询分区方法，而没有分析图形数据库独特的图形原生结构，因此无法全面测试图形数据库。

> [!NOTE] 相关工作论文笔记
> * [GDsmith：Detecting Bugs in Graph Database Engines 论文笔记_detecting transactional bugs in database engines v-CSDN博客](https://blog.csdn.net/qq_38135755/article/details/144200584)
> * Grand: [Finding Bugs in Gremlin-Based Graph Database Systems via Randomized Differential Testing | LinLi's Blog](https://linli1724647576.github.io/2024/03/01/Finding-Bugs-in-Gremlin-Based-Graph-Database-Systems-via-Randomized-Differential-Testing/)
> * GDBMeter: [Testing Graph Database Engines via Query Partitioning 论文笔记-CSDN博客](https://blog.csdn.net/qq_38135755/article/details/144200617?spm=1001.2014.3001.5502)


### 1.5 Overview of QuDi
1. **查询分解（Query Disassembling, QuDi）：一种有效的测试预言，用于揭示单个基于Gremlin的GDBs中的逻辑错误。**
2. **test oracle：一个复杂的Gremlin查询可以被分解为一系列原子图遍历操作（原子图遍历可以在图数据库中实现一步遍历），这些操作与原始查询具有等效的执行语义，因此应该获得相同的结果。**
3. **Insight：将Gremlin查询分解为一系列原子图遍历，可以防止某些优化策略的介入，从而检测图数据库中的逻辑错误。**
4. QuDi
	* 给定一个Gremlin Q，我们将其分解为一系列原子图遍历操作 TLiist = <T_1,T_2,...,T_n>, 其中T_i表示第i个原子图遍历。
	* 对于TList中的每个原子图遍历T_i，我们构建一个查询，该查询以其前一个遍历T_(i-1)的结果RS_T(i-1)作为输入，并计算其自身的结果RS_T(i)。
	* test oracle：查询Q的结果RS_Q必须等于TList中最后一个遍历T_n的结果RS_T(n)。提出了三种执行策略来实现这一测试预言，旨在发现更多的逻辑错误。
5. 实现评估
	* 测试GDB：基于 Gremlin 的图数据库（Neo4j[12]、OrientDB[16]、JanusGraph[8]、HugeGraph[6]、TinkerGraph[22]和ArcadeDB[14]）
	* 效果：25个逻辑错误。17个逻辑错误已被确认，其中10个被归类为先前未知的错误，7个被归类为现有重复错误。8个错误被GDB开发者修复。
	* 逻辑错误原因：17个已确认的错误中，11个错误是由原子图遍历组合的实现和优化不正确引起的，其余6个错误则是由原子图遍历的实现错误导致的。
6. 总结
	- 我们提出了一种通用（可适用于其他图数据库和关系型数据库皆）且有效的测试预言机——查询拆解，用于发现单个图数据库中的逻辑错误。通过将复杂的查询拆解成等效的原子图遍历序列，揭示与 Gremlin 查询在图数据库中实现和优化错误相关的逻辑缺陷。
	- 在六种广泛使用的图数据库上评估了 QuDi。总共发现了 25 个逻辑错误，其中 10 个已被图数据库开发者确认是之前未发现的错误。


## 2 Preliminaries
### 2.1 Labeled Property Graph Model（标记属性图模型）

![LogicBugInArcadeDB#500_graph](../Pictures/LogicBugInArcadeDB_500_graph.png)

* 图形数据库系统通常使用标记属性图模型(Labeled property graph model [44])来表示其图形结构。
* 该模型包含一组节点(nodes)和与这些节点相关联的一组边(edges)。
* 每个节点或边都有一个附加的标签(labels)，用于将其划分到特定的组中。同时，一组键值对属性(key-value pair attributes)用于描述节点或边的特性并提供额外的元数据。

### 2.2 Gremlin Query Language

7. 不同的图形数据库支持不同的图形查询语言。例如，Neo4j 开发了 Cypher，TinkerPop 开发了 Gremlin，TigerGraph 开发了 GSQL，ArangoDB 开发了 AQL 等。
8. **根据DB-Engines的图数据库排名，Gremlin是最受欢迎且广泛使用的图查询语言之一，大约半数的41个图数据库支持Gremlin。** 
9. **Gremlin**
	* 一种函数式语言（a functional language），遍历操作符被链接在一起形成类似路径的表达式。一个 Gremlin 查询由一系列 Gremlin 遍历原语组成，最终计算出最终的输出结果。
	* 示例：`g.V().where(values('name').is(eq('Alice'))).outE('created').inV().hasLabel('software').values('lang')`。该查询首先使用 `g.V()` 获取所有节点，然后使用 `where()` 和 `eq()` 过滤出名字为 Alice 的节点。通过使用 `outE('created').inV()` 遍历边，可以访问 Alice 创建的软件。最后，通过 `hasLabel()` 和 `values()` 返回软件的语言。在这个示例中，`where()` API 中存在一个嵌套查询，其中 `is()` 用于评估属性值是否与谓词匹配。

### 2.3 Gremlin Traversal Model（来自Grand）
![TraversalModelOfGremlin](../Pictures/TraversalModelOfGremlin.png)

10. Gremlin遍历模型：阐释如何构建有效的Gremlin查询。
11. 该模型的关键见解：查询中一个Gremlin API的输入类型应与其前一个Gremlin API的输出类型相匹配。
12. 模型说明
	* Vertex描述了返回顶点列表的操作。例如，Gremlin API V()可以从图数据库中检索所有顶点并返回一个顶点列表。
	* Edge代表返回边列表的操作。例如，可以使用g.V().outE()来检索图数据库中所有顶点的出边。
	* Filter是一组过滤操作。例如ℎas()和ℎasLabel()，它们映射满足给定过滤条件的实体。属于Filter的操作可以组装在Vertex和Edge之后，并分别返回顶点列表和边列表。例如，如果我们将g.V()和ℎas()组装在一起，组装后的查询g.V().ℎas()也会返回一个顶点列表。
	* Predicate包含一组谓词操作。例如用作Filter参数的lt()。
	* Value描述了返回值或值列表的操作。例如，我们可以通过g.V().values()检索顶点的属性。
### 2.4 Gremlin Query Execution
13. GRM：Gremlin查询由Gremlin遍历机（Gremlin Traversal Machine，GTM）处理。
14. GTM优化策略
	* 作用：旨在根据访问图数据的成本确定最优的执行计划。例如，优化策略可以重新排序过滤操作，以先执行成本较低的过滤器（即FilterRankingStrategy），或者合并操作以实现高效搜索（即IncidentToAdjacentStrategy）。

## 3 Approach
![QuDiOverview](../Pictures/QuDiOverview.png)

### 3.1 Graph Database and Query Generation





### 3.2 Query Disassembling


### 3.3 Atomic Traversal Execution



## 4 Evaluation

