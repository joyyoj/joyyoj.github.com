***************************
HiveQL简要说明
***************************

HQL可以粗略地分为两层：

* 算子层：操作数(expression）+操作符（Operator: select/join/groupby等）

  HIVE前端编译生成逻辑计划(算子树,Operator组成)后，再进行切割, 生成物理执行计划(MapTask/ReduceTask组成);

* 函数层：操作数(expression) + 操作符（UDF:case when, if, count, sum等)

  每个Operator的求值由若干个ExprEvaluator构成，如select a + 2，函数层需要add(column(a),2)

算子树执行的大概示意如下 ::

	Operator root.process() {
      // 操作数ExprNode求值，例如select a, b, c的a,b,c
	  // group by a+b, a的a+b,a
	  // ExprNodeDesc对应的求值器是ExprNodeEvaluator
	  Foreach(ExprNodeEvaluator eval : evals) {
		operand[i] = eval.evaluate();
      }
	  // 如果Evaluator是FuncEvaluator，则会再调用UDF来求值
	  ForEach(Operator child: children) {
		child.process(operand);
	  }
	}

操作数
=====================

（Expression使用ExprNodeDesc来序列化描述），主要包括：

1. ExprNodeColumnDesc 描述Column
2. ExprNodeConstantDesc 描述常量
3. ExprNodeGenericFunDesc 描述函数
4. ExprNodeFieldDesc 
   描述Field,HIVE支持复杂类型，主要用于描述struct的一个成员，如a.b;map和array的成员取值通过func实现。
5. ExprNodeNullDesc 描述Null


Operator
====================
