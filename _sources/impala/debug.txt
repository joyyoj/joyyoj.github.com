Impala常见问题总结
=======================

GDB调试产生大量的段错误
-------------------------------
  
impala be会使用jni调用访问fe的相关模块，当用gdb调试be 的时候，由于JVM垃圾回收会使用到一些信号，如果gdb调试时不自定义信号处理方法，则可能导致GDB直接退出。在实际调试中，JVM产生大量的Segmentation faults. 可以在gdb执行run之前，添加命令 ::

  gdb> handle SIGSEGV nostop noprint pass
  
此外，类似的gdb配置选项还有 ::

  handle SIGSEGV pass noprint nostop
  handle SIGUSR1 pass noprint nostop
  handle SIGUSR2 pass noprint nostop

.. gdb bin/start-impalad.sh -use_statestore=false
.. gdb -q impalad
.. set args -use_statestore=false -nn=hadoop-01.localdomain -nn_port=8030


