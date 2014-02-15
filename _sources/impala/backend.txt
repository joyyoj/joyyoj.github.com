***************************
Impala后端读取文件的实现
***************************

Impala的系统架构
=======================

be代码结构
================

* service: 连接前端，并接受client的请求

* runtime: 运行时需要的类，包括coordinator, planfragment,datastream, mempool, tuple等

* exec: 主要是ExecNode及子类，还包括读各种文件的Scanner，目前Scanner还没有分离出读文件的和解析内容两层；

* exprs: 各种表达式，如tableref，case等。目前表达式没有抽象提取出函数，因此新增一个函数就需要新加一个expr；

* transport: Thrift SASL: Simple Authentication and Security Layer

* statestore: statestore, simplescheduler, statestoresubscribe等；

* codegen: 代码生成

每个impalad都可以处理Query请求，而其他的impalad作为Backend，在实际执行时，
  每个impalad都可以作为接收Query，其他Impalad都是一个Query包含由Coordinator和多个PlanFragmentExecutor共同完成。
  PlanFramgent会发到Backend上执行


Impalad的启动流程
=======================

* 初始化LLVM，hdfs，jni，hbase，前端;
* 启动ImpalaServer
* 启动thriftserver，接受thrift请求
* 启动ExecEnv
* 启动webserver
* 启动SubscriptionManager
* 启动Scheduler
* 向StateStore订阅，并注册回调函数SimpleScheduler::UpdateMembership，用于调度
时提供当前可用的后端
SubscriptionManager::RegisterService
StateStore检查service是否存在，如果不存在，则建立一个新的serviceinstance
检查客户端是否存在于这个serviceinstance的membership中，如果不存在，则添加一个
SubscriptionMangaer::RegisterSubscription
StateStore添加一个Subscriber，订阅这个service的membership，并注册回调函数
MembershipCallback
当有update回调时，更新impalaserver
的membership状态，用于failure detector
Impalad启动后，就可以接受query请求，也可以接受其他impalad的请求，执行一个PlanFragment


Coordinator::Exec
==============================

1. ComputScanRangeAssignment;

2. ComputeFragmentExecParams;


为什么需要Conjuncts？而Expr不够表达？
--------------------------------------

CreateConjuncts


====================
  PlanFragmentExecutor::prepare {
    设置desc_tbl;
    设置plan_;
    plan_->Prepare;
    设置ExchangeNode的sender数目
    对ScanNode设置ScanRange;
    设置data sink;
    设置profile;
  }
  PlanFragmentExecutor::OpenInternal {
    plan_->Open(); 
    while (true) :
      GetNextInternal {
        while undone_:
          plan_->GetNext(); 
      }
      sink_->Send;
    sink_->Close;
  }

Impala的Cache策略
======================

TUniqueId fragment_instance_id,
PlanNodeId dest_node_id,

StreamControlBlock
