***************************
Impala IO/数据流分析
***************************

总体流程
=============================

当用户执行Query时，Query被翻译成执行计划，物理执行计划其实就是ExecNode组成的DAG图。这其中扫描数据由ScanNode完成。
节点之间交互数据由ExchangeNode完成;

在节点之间交互的数据流采用推的方式，具体而言:

* 每一个PlanFragmentExecutor的计算节点完成计算后，通过DataSink主动将数据下推给下游节点；

* impalad的ImpalaServer实例收到DataStream类型的数据包时，调用DataStreamMgr::AddData()写入数据. 根据fragment_id/node_id分发给它管理的DataStreamRecvr,ExchangeNode等用户通过DataStreamRecvr接收数据。

* ExchangeNode在Prepare时创建DataStreamRecvr;
  ExchangeNode在GetNext时会调用DataStreamRecvr的GetBatch(阻塞操作)

Impalad底层的读取文件的IO模型是典型的生产者-消费者模型,Scanner是消费者，而DiskIoMgr线程是生产者.

* Impalad在初始化环境时,将新建DiskIoMgr对象,并启动多个IoMgr线程(默认一块Disk一个IoMgr线程);
  IoMgr线程,从磁盘扫描数据;

* ScanNode调用Open时,负责将要读取的ScanRanges Add到DiskIoMgr中;并启用
  若干个Scanner线程读取数据. Scanner线程将调用GetBytes(阻塞), 然后处理bytes，物化Tuples;

* DiskIoMgr的线程在发现ReaderContext非空后，按照Round-Robined算法调度ReaderContext，读取数据;

Impala与HIVE的数据流对比
-------------------------

* HIVE的每个Task内部执行时，算子树的求值是推的方式。如MapTask先执行MapOperator，通常是先TableScanOperator，
  然后将数据分给下游，如TableScanOperator->FilterOperator->SelectOperator->GroupbyOperator->ReduceSinkOperator；
  然后ReduceSinkOperator将数据输出给MapReduce框架，MapReduce框架在执行时，ReduceTask主动去拉取MapTask的输出。

* 在Impala中，PlanFramgent相当于HIVE的一个Task，在PlanFrament的算子树求值时，下游节点调用GetNext向上游节点拉取，
  而PlanFragment执行完一个GetNext之后，就会把通过DataSink方式将RowBatch推送到其他PlanFragment；

总结起来HIVE是算子树内部求值是推的方式，Task之间是拉的方式。而Impala则算子树求值是拉的方式，PlanFragment之间是推送的方式。
这样的优势我能想到两个明显的优势:

* 利用带宽更加有效;

* 对于select * from table limit 10这样的语句，
  HIVE需要等待整个HadoopJob执行完成之后才可以输出结果，Impala只要Coordinator拿到10条记录后就可以Cancel掉其他后端。
  

Impala底层读取IO的详细流程
==============================

ImpalaServer在初始化环境时，创建ExecEnv对象 ::

  ExecEnv的构造函数 {
    新建各种对象，与IO相关的，主要包括DataStreamMgr,DiskIoMgr;
    DiskIoMgr的构造函数 {
      设置每个disk对应一个DiskQueue;
      设置每个disk读取的线程数目为FLAGS_num_threads_per_disk;
    }
    调用disk_io_mgr的Init() {
      For 每个disk_queue:
        构造DiskQueue对象;
        每个DiskQueue启动num_threads_per_disk个线程;
        
      线程的主循环是ReadLoop :{
        while (true) {
          调用GetNextScanRange获取下一个要读的ScanRange: {
            DiskQueue采用Round-Robined轮转算法，取出队首ReaderContext;
            获取ReaderContext的要读取的ScanRange;
            判断进程是否超内存限制;
            判断Reader是否超内存限制;
            If 进程超内存限制,调用GcIoBuffers();
            If 依然超内存限制，就Reader->Cancel()
            If reader是Canceled状态: 
              continue;
            If reader下一个ScanRange为空并且unstarted_ranges队列非空:
              将reader的unstarted_ranges队列取出一个ScanRange放入到>
              ready_to_start_ranges_队列; 
              // 这样做的主要目的在于节省GetNextScanRange等待时间
            唤醒一个阻塞在读GetNextScanRange的线程;
            将reader重新放回disk_queue的队列;这样线程可以处理其他ScanRange
          }
          GetFreeBuffer(reader);//分配max_reader_size_ byte
          GetBufferDesc;
          range->OpenScanRange //打开文件，然后fseek到要读取的offset;(如果hdfs_connection_为空，是本地读取)
          更新Counter;
          range->ReadFromScanRange //从打开的文件句柄中读取数据;
        } // while true
      } 
    }//Init
  }

HdfsScanNode
==============================

::

  Prepare
    tuple_desc_=state->desc_tbl().GetTupleDescriptor();

    collect all materialized (partition key and not) slots
    // Finally, populate materialized_slots_ and partition_key_slots_ in the order that 
    the slots appear in the file.  
    Codegen Scanner需要的函数
    设置当thread可用时启用的callback函数HdfsScanNode::ThreadTokenAvailableCb,至少要求一个线程
  ThreadTokenAvaliableCb的实现 {
    只要有线程可启用，就启动Scanner线程。不过以下几种情况不用启动新线程:
    1. ScanNode已经完成
    2. 所有的Ranges已经被其他线程拿走 (?)
    3. 剩余的Ranges < 活跃的线程数 
    线程的主循环 {
      while (true) {
        disk_io_mgr获取下一个ScanRange;
        创建新的Scanner对象来处理这个ScanRange;
        scanner->Prepare;
        scanner->ProcessSplit;
        scanner->Close;
      }
    }
  }

  //Open时初始化hdfs的连接，同时会启用Scanner线程. Scanner将向DiskIOMgr的队列添加Splits
  Open(RuntimeState* state):
    向disk_io_mgr注册Reader，传递hdfs_connections
    调用ScanRange的每个Partition的PrepareExprs
    初始化所有Scanner的Ranges


BaseSequenceScanner
======================
::

  ProcessSplit:{
    //对于每一个ScanRange,需要InitNewRange, ProcessRange
    header = scan_node_->GetFileMetadata();
    if (header_ == NULL)
      ReadFileHeader(); // parsing header
      scan_node_->SetFileMetadata();
      scan_node_->AddDiskIoRanges();

    InitNewRange;
    // 第一条记录
    if stream_->scan_range->offset() == 0 :
      skip header;
    else
      skip to sync

    do {
      //处理当前的Range直到出错或者结束
      //[要求在Data block的开始位置，这个通过sync marker来保证]
      ProcessRange() {
        while (!eof) {
          GetRecord;
          GetMemory;
          如果当前slot需要物化,解析当前Record
          否则，直接写入
        }
      }
      如果不允许错误，退出;
      否则,跳到下一个Sync
    } while (!stream_->eosr());
  }
