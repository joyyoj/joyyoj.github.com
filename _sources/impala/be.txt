****************************
Impala后端分析
****************************


查询后端主要class介绍
=======================

ImaplaServer
---------------------------

ImpalaServer是impalad运行时所有请求的入口，以TExecRequest为例，由ImpalaServer::QueryExecState 负责处理。TExecRequest中包含TQueryExecRequest，描述了Frontend解析SQL后，向Backend发送的查询请求。

ImpalaServer::QueryExecState包含以下方法:

1. ImpalaServer::QueryExecState::Exec(TExecRequest* exec_request):
 
对TQueryExecRequest进行基本检查，然后调用Coordinator的Exec()方法解析提交请求

2. ImpalaServer::QueryExecState::FetchRows(const int32_t max_rows, QueryResultSet* fetched_rows)

先调用Coordinator::Wait()，阻塞等待，该调用结束，意味着若干查询结果已生成，通过ImpalaServer::QueryExecState::FetchNextBatch()调用Coordinator::GetNext()，获取一个小批量的结果，并按自己的处理能力，分批处理这些结果。

Coordinator
---------------------------

1. Coordinator::Exec()

Coordinator在Exec(const TUniqueId& query_id, TQueryExecRequest* request, const TQueryOptions& query_options)方法中，根据收到的query_id和request，会计算产生一次查询相关的参数信息，保存在本地。

1. 记录query_id，保存在Coordinator::query_id_
2. 从request中取出finalize_params，保存在Coordinator::finalize_params_
3. 从request中取出query_globals，保存在Coordinator::query_globals_
4. Coordinator::ComputeFragmentExecParams(const TQueryExecRequest& exec_request) 根据request计算出每个PlanFragment的执行参数，保存在Coordinator::fragment_exec_params_中
    4.1 Coordinator::ComputeFragmentHosts(const TQueryExecRequest& exec_request) 对每一个PlanFragment，根据request中输入数据所在的位置(per_node_scan_ranges)，得到执行查询的impalad的Host列表，保存在Coordinator::hosts中
        4.1.1 从PlanFragments的DAG的底层向上计算每个PlanFragment的执行节点。因为有些PlanFragment可以直接使用其子Fragment的执行节点执行。
            a) 若PlanFragment的partition.type == TPartitionType::UNPARTITIONED，说明它只能在一个impalad上执行，把它放在coordinator上执行
            b) 称PlanFragment.plan按DFS序执行第一个被执行的节点（最左子节点，或根节点）为LeftmostNode，若LeftmostNode的查询类型是EXCHANGE_NODE（用于收集DAG下层的PlanFragment的数据)，且所有PlanFragment中存在一个PlanFragment，向当前LeftmostNode输出数据，则选择那个PlanFragment所在的hosts列表作为当前PlanFragment的执行节点。若LeftmostNode的类型是HDFS_SCAN_NODE或HBASE_SCAN_NODE，从LeftmostNode的scan_range_locations指定的一堆DataNode中，若DataNode上存在impalad，则选择该impalad执行，否则从所有impalad中，用Round-robin（轮询）算法选择一个impalad节点执行。在data_server_map中记录每个DataHost上的数据，由哪个impalad处理。
    4.2 为4.1生成的hosts列表中每个THostPort生成一个全局唯一的instance_id。
    4.3 统计每个PlanFragment的输入源,记录向该PlanFragment中的EXCHANGE_NODE写入数据的PlanNode的instance_id,及向该PlanFragment写入数据的hosts的数目. 目前，StreamSink的type只能是TPartitionType::UNPARTITIONED,即PlanFragment只能通过广播的形式，将所有数据发送给所有下游PlanFragment。
 5. Coordinator::ComputeScanRangeAssignment(const TQueryExecRequest& exec_request) 对每个PlanNode所需扫描的Range，计算扫描量，安排负责执行的物理节点
    5.1 对于scan_range_locations中每个TScanRangeLocation，根据scan_range计算大概的扫描量，从locations中选择目前扫描量最少的location，作为负责扫描该range的DataNode，并更新它的扫描量。
    5.2 从data_server_map中，找到负责扫描该DataNode的impalad，记录scan_range和impalad的对应关系
6. 通过ParallelExecutor并行调用Coordinator::ExecRemoteFragment(void* exec_state_arg)
    6.1 通过impala-client，向每个PlanFragment所在机器发送请求，执行机根据请求构建PlanFragmentExecutor
    6.2 调用PlanFragmentExecutor的Prepare()方法，初始化执行过程
        6.2.1 各种初始化
        6.2.2 通过LLVM生成执行代码 
        6.2.3 设定扫描的Range，输出数据流等
7. 初始化ProgressUpdater progress_，后续用于收集执行状态
4.4.2.2 Coordinator::Wait():
Coordinator::Wait()中，调用PlanFragmentExecutor::Open()方法，阻塞等待远程的impalad返回。如果PlanFragmentExecutor中定义了sink，则Open()方法中即不断循环调用PlanFragmentExecutor::GetNext()，流式地获取数据，通过sink发送到下游。Coordinator::GetNext()调用PlanFragmentExecutor::GetNext()方法，并根据本机的处理能力，对流式返回的数据量做一些限制及意外处理。PlanFragmentExecutor::GetNext()每次从PlanNode中调用GetNext()获取一批数据，返回外层。

ExecNode
-------------------

ExecNode是各种PlanNode的基类，提供一些初始化、析构等公共方法，还有CreateTree方法，根据TQueryExecRequest中的TPlan，构建PlanNode Tree。

目前PlanNode有7种：HDFS_SCAN_NODE、HBASE_SCAN_NODE、AGGREGATION_NODE、HASH_JOIN_NODE、EXCHANGE_NODE、SORT_NODE、MERGE_NODE，他们组合成PlanNode Tree，完成各种复杂的查询操作。

DiskIoMgr
----------------------

DiskIoMgr 负责管理,调度所有IO请求.每个PlanNode会申请一个或多个ReaderContext，每个负责一系列的ScanRange.有独立的队列。
PlanNode通过DiskIoMgr的API，获得ScanRange(non-blocking)和读取数据(blocking)。DiskIoMgr有自己的工作线程，从本地磁盘或者HDFS读取数据。
DiskIoMgr通过ReaderContext实例标识一个请求，用户通过RegisterReader()/UnregisterReader()申请、释放ReaderContext实例，通过GetNext()/Read()等方法操作ReaderContext实例，获取数据。

DiskIoMgr可以理解成一个线程安全的生产者消费者管理器，管理两个层面的资源：

1. 硬件资源：读数据的Reader实例的队列；可复用的空Buffers队列

2. 读取请求：描述查询请求的ScanRanges队列；已使用的Buffers队列

磁盘工作线程，不断从空Buffers队列中取出Buffer，从ScanRanges队列中取出要处理的ScanRange；处理完成后，加入已使用的Buffers的队列。
用户通过AddScanRanges()向ScanRanges队列插入新的ScanRange请求；通过GetNext()消费写有数据的Buffer；通过ReturnBuffer()将用完的Buffer返回空Buffers队列。

DataStreamMgr
------------------
DataStreamMgr: 负责管理所有PlanNode间的输入输出流. 
impalad的ImpalaServer实例收到DataStream类型的数据包时，调用DataStreamMgr::AddData()写入数据，它根据fragment_id/node_id分发给它管理的DataStreamRecvr，ExchangeNode等用户通过DataStreamRecvr接收数据。

ScanNode 
------------------

HdfsScanNode
^^^^^^^^^^^^^^^^
 HdfsScanNode负责从HDFS读取数据，反序列化，它启动一个DiskThread用于读取二进制数据，启动多个ScannerThread反序列化这些数据。Scanner是反序列化HDFS文件的实例，目前支持Txt、RCFile等各种格式，增加新的Scanner，可以支持OLAPEngine等。

::

  Open()

      GetDefaultConnect() // 建立HDFS的连接，初始化统计信息，

      RegisterReader() // 初始化ReaderContext实例

      add_thread(DiskThread())

      IssueMoreRanges() // 初始化新的ScanRanges
          
  DiskThread():
      io_mgr()->GetNext() // 获取一个已经写有数据的Buffer
      // 找到获得的Buffer对应的ScanRange的Scanner
      StartNewScannerThread(): // 启动Scanner线程读取Buffer
              CreateScanner() // 根据HdfsPartition类型构造HdfsScanner
              add_thread(new thread(ScannerThread))

  ScannerThread():
      scanner->ProcessScanRange() // 调用Scanner处理Buffer中的信息

      if 当前所有ScanRange都已完成:
          IssueMoreRanges() 向DiskIoMgr添加新的ScanRange请求

AggregationNode
------------------------

基于Hash的内存聚集，利用LLVM生成算子计算Hash值和聚集结果。

HashJoinNode
--------------------------

支持INNER JOIN / LEFT OUTER JOIN / RIGHT OUTER JOIN / FULL OUTER JOIN / SEMI JOIN，但JOIN条件必须包含至少一个等值JOIN，即不支持Cross Join。

HashJoinNode会有两路输入（分别称为左路和右路），每次HashJoinNode::Open()方法中，会先流式拉取右路的所有输出结果，存在内存中。然后从左路中流式拉取数据，对内存Hash表中的右路数据进行匹配。根据JOIN类型输出JOIN结果。

ExchangeNode
-----------------------------

负责接收其它Fragment的PlanNode的输出（可能跨网络）。Prepare()方法中，从DataStreamMgr中获取DataStreamRecvr的实例，在GetNext()中不断从DataStreamRecvr取数据。

MergeNode
-------------------------------
合并Fragment内所有上游PlanNode输出的结果。合并过程中，会根据每个PlanNode的Schema对数据进行Schema Change，输出成统一Schema的结果。 

