HIVE实现Hst支持的详细设计文档
===============================

实现HIVE直接读取HST的字段
=============================

HST外层使用PB，但是有一个字段是binary，是linkbase自己的格式dlb_link_attr_t，所有的binary都是固定的struct;因此还是需要支持特殊的Serde，但是可以复用ProtobufDeserializer，只是需要特殊处理的是如果是binary，对应的ObjectInspector要修改为DlbLinkAttrObjectInspector，令其能够读取这个Struct;

需要的工作：
1 实现DlbLinkAttrObjectInspector,用来解释binray。有两种实现：直接可以通过jni调用已有的反序列化函数，将binray转换成\t分割的字符串;或者是java重新实现一遍，前者容易维护，开发代价小，后者开发代价较高，但性能较高。

方案：先通过jni调用，快速实现HIVE读取HST;

预估时间：4人天; 

实现select * from table where key>=100的高效优化;
===========================================================

由HiveStorageHandler下推Filter，将key相关的条件=, >, <, >=, <=下推到TableScan，由相应的InputFormat处理Filter;
谓词下推后，由于语句将变成TableScan->Select的方式，因此直接可以使用fetch优化,由本地Fetch

需要的工作：
1 编译器实现谓词下推, 大体类似目前推BucketPruner下推Filter,不同之处:比如residual需要去除相应的filter，还有具体下推的类型不一样;
2 实现能处理Filter的HSTInputFormat; 
3 由于FetchOperator在实现时，迭代输入文件，计算splits,这一步其实可以进一步优化，一方面getSplits最好只返回符合条件的Split，另外许多文件其实没有必要再getSplits;
4 进一步改进Fetch, 优化TableScan->Select->Select也为本地Fetch;

预估时间：
1 谓词下推，2人天;
2 提供读取Index的接口，2人天;
3 实现Filter的InputFormat; 3人天;


HstHiveInputFormat的实现
==========================
实现
HstTextInputFormat;
HstBalancedSequenceFileInputFormat;
将信息下推到JobConf; 

JobConf.GetTextInputFormat("IndexHandler");
getSplits


如何利用Sorted信息，高效地实现join?
====================================

方案一
-------------------------------------

由于参与join的所有HST表全有序，此时在Map端使用SortedMerge Join实现方式性能优于HashJoin的实现。
为了方便处理，由用户提供mapjoin的hint，同时设置hive参数sortedmerge=true;
HIVE的内部实现方式类似SortedMergeBucketMapJoin，需要几个步骤:

1 遍历Task DAG，更新执行计划;
主要内容: 实现SortedMergeMapJoinOptimzer的逻辑算子树的Transform, Transform规则把MapJoin给替换成SortedMergeMapJoin;

2 MergeSortJoinOperator在Map端完成，TableScan大表，小表在MapTask处理时远程读取;

具体实现的逻辑类似SMBJoinOperator,不过FetchOpeator的实现不同.

优势:
1 改动代码较小，可以较快实现;


方案二
------------------

主体思路：将Join算子的实现下推到InputFormat, 调用Reader的next时，返回的值就是join后的一条记录。
例如：hql: A JOIN B JOIN C ON A.key = B.key and B.key = C.key;
其中A是大表。将JOIN B JOIN C的实现完全下推到InputFormat, 在算子层看不到B和C的存在，当调用的Record时返回的key,value就已经是join后的结果;

需要的工作：

1 约定配置JOIN需要的joinkey，jointable的方式;
2 实现merge sort join的InputFormat;
3 更改执行计划，需要替换join算子;

优势:
InputFormat更底层，因此HIVE和HST分布式数据处理框架都可以使用;

存在的问题:
1 实现代价高，执行计划的改动方案不是非常明确，有一定的不确定因素;
2 支持全部JOIN算子(..., left outer join, right outer join...)比较困难;

由于两个方案的主体代码可以复用, 可以先实现方案一，在这个基础上重构，实现方案二;

时间预估：
1 方案一：2-3周
2 方案二：2周

算法的实现 ::
    
对于mapjoin a, b，则把MapJoinOperator去掉，同时设置

  class SortedTable {
  int SeekToKey(Object key);
  Object next();
  boolean hasNext();
  };
  BRecordList bRecords;

  processOp(Object row, int alias) {
  aKey = GetJoinKey(row, alias);
  if (bRecords.key() == aKey) {
  while iterater bRecords:
  output join results to children;
  return;
  } else {
  bRecords.clear();
  }
  B.SeekToKey(aKey);
  // build bRecords
  while (B.hasNext()) {
  [bKey, bValue] = B.next();
  if (bKey == aKey) {
  bRecords.put(bValue);
  } else if (bKey > aKey) {
  return; 
  }
  }


细节问题
==================

如何访问IndexServer的接口？
----------------------------
1 访问IndexServer获取得到某个Key最开始的位置;

HIVE的JOIN逻辑
==================

for (i = 0; i < numAliases; ++i) {
  RowContainer<ArrayList<Object>> alw = storage.get(alias);
  if not outer join:
    if alw is empty: return;
    else mayHasMoreThanOne = true; 
  else :
    if alw.empty() : 
      hasEmpty = true;
      插入一条dummyObject;
}
// if outer join and alw.size() > 0
// inner join and all table has only one key
if all not empty and has only one:// result should be only one
   genAllOneUniqueJoinObject(); 
elseif all not empty and not left semi join:
   genUniqueJoinObject(0, 0);
else: 
   genObject; 

// outer join, and may be some all null
A 3 key; B 4 key; C 1 key;
3 * 4 * 1
result should be: 
for (Object a: A) {
  for (Object b: B) {
    for (Object c : C) { 
    }
  }
}


convertToSMBJoin
==================

smbJoinDesc.setTagToAlias();
1. Check wheth table is global sorted
1. convert a bucket map join operator to a sorted merge map join operator
1.1 removeMapJoin;
1.1 addSMbJoinOperator;


hive.auto.convert.join = true;

CommonJoinDispatcher
选择bigtable的alias; 根据文件的大小来判断;

select * from src a join src b on a.key = b.key where a.key > 100;

initializeOp()

要求可以getSplits时进行过滤;
实现一个HstHiveRecordReader,可以seek;
实现一个HstHiveInputFormat，返回HstHiveRecordReader;
HstHiveInputFormat; HstHiveOutputFormat;

谓词下推的实现逻辑
--------------------

If C1 AND C2:
   PUSH C1.pushed and C2.pushed;
   RES C1.reserved and C2.reserved;
ELSEIF (C1 OR C2) AND (C1.RES is NULL AND C2.RES is NULL):
   PUSH C1.pushed or C2.pushed
   RES NONE;
ELSEIF C1 is typeof  SortedColumnA (><>=) Constants:
   PUSH C1;
   RES NONE;
ELSE:
   PUSH NONE;
   RES ALL;


// determinstFunc(bucketColumn) CompareOp determinstFunc(constants)
// sortedColumn CmpOp determinstFunc(Constants)

//InputSplit[] inputSplits;
class HstTextInputFormat extends TextInputFormat implements InputFilterAvaliable {
   getRecordReader() {
    //pushProjectionsToFilter;
    Next() {
      GetKeyRange;
      执行getNext时
      // getFileName From Conf;
      // getKeyRange From Conf;
      IndexHandler indexHandler;
      indexHandler.open(indexFileName);
      indexHandler.seek(keyRange);
      while (indexHandler.next(IndexRecord record)) {
        record.seekToPosition(record.filename); 
      }
      seekToFirst
    }
    //return null?
  }
}
 
getSplits() {
  Get InputPaths;
  foreach path:
    Get PartitionDesc of path;
    pushFilterToJobConf;

    ConfUtil.GetKeyRange(path, conf);
    check whether path is pass filter;
  super.GetSplits();
}


CombineHiveInputFormat:

how to implements getSplits ?

interface PathFilterCapability {
   void prune() {
   }
};


//下推的条件中，只下推key >=< 常量的表达式;
IndexHandler目前只处理SortedKey >=< 常量的表达式;
