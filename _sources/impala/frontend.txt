*********************
Impala前端分析
*********************

FrontEnd Overview
=====================

frontend 指的是翻译 query 语句,生成执行计划的部分,采用 java 实现; 通过 jni与c++代码交互.
Frontend 主要完成的工作包括 Query 的语法分析,语义分析,生成基本查询计划树 (PlanNode 树) ,生成 PlanFrangent 及之间的相互关系.其他包括获取相关元数据信息,如所有表 scan node 的 location,select 语句结果字段的元数据等.


SQL解析
=======================
Impala使用jflex生成词法扫描器，词法定义文件为sql-scanner.flex；使用cup生成语法解析器，语法定义文件为sql-parser.y.

主要流程
---------------
1. 完成词法语法解析，生成语法树；
2. 构造查询分析器，对语法树进行分析，生成描述查询的内部数据结构，对于select而言，描述查询的数据结构主要是Analyzer和SelectStmt；

Frontend类的createExecRequest函数，主要功能是根据client的查询请求，生成TExecRequest，是前端代码的总入口，其主要执行流程如下：

1. 创建AnalysisContext，执行analyze函数完成sql词法、语法、语义解析；

2. 调用planner.createPlanFragments函数，生成所有的PlanFragment；

3. 设置每个fragment的destination；

4. 为每个scan node按照数据物理位置分布以及最大扫描字节数进行切分，生成一组scan range；

5. 根据查询结果列信息，填写TExecRequest的result meta；

Planner
============================
Planner负责将分析后的语法树转换成执行计划，即PlanFragments。

Planner的入口函数是Planner::createPlanFragments()。其主要流程如下：
createSinglePlan
-------------------------
根据语法分析的结果，遍历并生成一个逻辑执行计划树。 树中的节点是PlanNode。树的根
节点是最后被执行的节点。

createPlanFragments
-----------------------

将生成的执行计划切割成多个PlanFragment。 这个过程类似Hive中将执行计划分割成多个
MapRed阶段。
目前Impala划分阶段的规则是： 如果遇到ScanNode, HashJoinNode, MergeNode, AggregationNode, SortNode，则产生一个新的PlanFragment。 每一个PlanFragment 之间通过ExchangeNode相连。ExchangeNode在运行时中会负责跨Impalad的数据传输。

computeMemLayout
--------------------------

对于输入表，计算在内存中的layout。 主要目的是为了计算一条记录在内存中会占用多少字节（由TupleDescriptor描述）， 以及每一个字段在内存中如何分布（由SlotDescriptor描述）。
最后PlanFragments就会成为物理执行计划，转成thrift后返回给前端。

逻辑计划生成
-------------------------------

代码请参考Planner.java的createSelectPlan函数。

逻辑计划最终生成结果是一颗由PlanNode子类组成的查询计划树，其生成流程为：
1. 从from语句里最左边的表开始，逐级自底向上构建一颗hash join的二叉树，最左边的表是这颗二叉树的最左叶子节点，root指针指向二叉树的根节点
2. 根据SelectStmt里的aggInfo，在root指向的节点之上增加一个AggregationNode节点，并把root指向新节点
3. 根据SelectStmt里的SortInfo，在root节点之上增加SortNode，并把root指向新节点
4. 返回root


物理计划生成
-----------------------------

代码参考Planner.java的createPlanFragments函数。

物理计划生成的结果是一组PlanFragment，执行是一个递归过程：

1. 首先根据root的子节点，创建对应的childFragment

2. 根据root的类型，对childFragment进行修改或合并操作

其中，第2步是关键，一些值得一提的规则如下:

1. 如果root是ScanNode，则创建一个只包含scan操作的PlanFragment

2. 如果root是HashJoinNode，则为第二个childFragment创建一个ExchangeNode用来转发数据，修改root的右子节点的PlanNode为以上的ExchangeNode，把修改后的root作为第一个childFragment的PlanRoot，返回第一个childFragment

3. 如果root是AggregationNode，则为childFragment的PlanRoot增加以上的AggregationNode，用于完成local aggregation，然后新增一个PlanFragment用于做global aggregation

4. 如果root是SortNode，判断childFragment是否是一个汇聚节点，如果是，则直接在childFragment的PlanRoot上添加SortNode节点；如果不是，则增加一个PlanFragment，内含一个ExchangeNode，用于汇聚childFragment的数据，然后在ExchangeNode之上添加SortNode节点，作为新增PlanFragment的PlanRoot.

子查询支持
-------------------------

Impala可以支持子查询,支持的方式也比较简单.一般情况下,子查询都被当做是另一种类型的表,子查询和普通表分别用InlineViewRef和BaseTableRef来表示,它们都是TableRef的子类.
在做语法解析的时候,子查询被当做是TableRef的一种,与其他普通表的TableRef一起保存在SelectStmt的List<TableRef> tableRefs里.
因为子查询有自己的SelectStmt和Analyzer, 在做逻辑计划生成的时候,子查询会先完成自身查询计划的构建,生成一颗子树,然后挂接到最外面查询的树上,共同构成一颗完整的查询计划树.这样后面在做物理查询计划生成的时候,就不需要关心子查询的事情.


JOIN处理
------------------------
目前impala只支持一张大表与多张小表之间的hash join，而且每个小表和大表之间至少包含一个等值join关系。由于还不支持根据表的元数据来判断表的大小，所以当前在写SQL的时候，必须人为的把大表写在from的最左边。实际执行的时候，在每一台对大表进行scan的服务器上，先把所有的小表装载到内存，并按join key算hash形成hash table，然后每次scan一行大表的数据，就会按照SQL里表出现的顺序从左到右依次与小表完成join关系运算，由此实现hash join的分布式化。

