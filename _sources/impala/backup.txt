ScannerContext
-------------------

这个类抽象了从IoMgr获取buffer的逻辑。每个实例被一个线程使用，因此非线程安全。
每个HdfsScanner对象一个ScannerContext,而ScannerContext包含1个或者多个ScannerContext::Stream(对于列存储的文件，每个ScannerContext包含多个Stream), 每个Stream包含1个ScanRange。
每个ScannerContext对应处理一个hdfs split

与ScannerContext交互的主要有3个线程:

DiskQueue
==============================

每个DiskQueue对应一个diskid,内部维护一组排队的ReaderContext.
ReaderContext

// Internal per reader state. This object maintains a lot of state that is carefully
// synchronized.  The reader maintains state across all disks as well as per disk state.
// A scan range for the reader is on one of four states:
// 1) PerDiskState's unstarted_ranges: This range has only been queued
//    and nothing has been read from it.
// 2) ReaderContext's ready_to_start_ranges_: This range is about to be started.
//    As soon as the reader picks it up, it will move to the in_flight_ranges
//    queue.
// 3) PerDiskState's in_flight_ranges: This range is being processed and will
//    be read from the next time a disk thread picks it up.
// 4) ScanRange's outgoing ready buffers is full. We can't read for this range
//    anymore. We need the caller to pull a buffer off which will put this in
//    the in_flight_ranges queue. These ranges are in the ReaderContext's
//    blocked_ranges_ queue.
//
// If the scan range is read and does not get blocked on the outgoing queue, the
// transitions are: 1 -> 2 -> 3.
// If the scan range does get blocked, the transitions are
// 1 -> 2 -> 3 -> (4 -> 3)*

TODO
