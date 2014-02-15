*************************
HIVE源码分析之编译（一）
*************************

前端编译的主要组成
########################

HIVE使用Antlr生成抽象语法树。Antlr主要包括：
* Lexer： 文法分析器类。主要用于把读入的字节流根据规则分段。
* Parser： 解析器类。主要用于处理经过 Lexer 处理后的各段。一些具体的操作都在这里。 


对应的代码为ql,主要包括:

- Compiler
- Parser
- TypeChecking
- Semantic Analysis
- Plan generation
- Task generation
- 执行引擎
- Plan

主要流程:

* 生成AST抽象语法树；
* 生成QB Block；
* 生成逻辑算子树；
* 逻辑算子树的优化；
* 生成物理的执行计划；
* 物理执行计划的优化；

compile
===================

genPlan (QB qb)
----------------------
1. 递归遍历子查询，生成子句的~plan，映射子句与对应的~OP；
2. 遍历~Table~，生成~TableScan~算子，映射表与对应的~OP；
3. genLateralViewPlans
4. 处理~JOIN~算子
	- 如果是~UNIQE JOIN，？
    - mergeJoinTree
5. genBodyPlan

genBodyPlan
^^^^^^^^^^^^^^^^^^^^^
then
called
to
handle
WHERE-­‐
GROUPBY-­‐ORDERBY-­‐SELECT
clauses.
– genFilterPlan()
for
WHERE
clause
– genGroupByPalnMapAgg1MR/2MR()
for
map-­‐side
par:al
aggrega:on
– genGroupByPlan1MR/2MR()
for
reduce-­‐side
aggrega:on
– genSelectPlan()
for
SELECT-­‐clause
– genReduceSink()
for
marking
the
boundary
between
map/
reduce
phases.
– genFileSink()
to
store
intermediate
results

常见的查询语句
=========================

**select** 

	select * from src;
	--包含查询子句
	select a from (select key as a from src) tmp;

**limit**

	select k1 from src limit 1;

**where**

	select k1, k2 from src where key=1;

**实例分析1**

	select k1 from src where key > 2 limit 1; 

* 生成抽象语法树：

![image](/images/hive/)


**实例分析2**

查询子句如何处理？

	select k from (select key as k from src);

**union all**

	select * from (
	  select key, value from src1 union all select key, value from src2
	) tmp;

**distinct**

	select distinct key from src;

**group by**

	select count(*), count(1) from src limit 1; 
	select key, count(*) from src group by key;
	select key, count(distinct value) from src group by key having key>2;

**join**

	select src.key, src.value from src join src_tmp 
	on src.key = src_tmp.key 
	where src.value="bcd";

### Antlr
### 
