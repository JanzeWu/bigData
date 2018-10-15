# 调整Spark工作

spark.executor.cores 每个 Spark 执行程序的内核数（执行器使用的内核数）。

spark.executor.memory 当 Hive 在 Spark 上运行时每个 Spark 执行程序的 Java 堆栈内存的最大大小（执行器使用的内存）。



## Spark是如何执行程序的
一个Spark应用包括一个driver进程和若干个分布在集群各个节点上的executor进程。

driver主要负责调度一些高层次的任务流（flow of work）。 executor负责执行这些任务，这些任务以task的形式存在，同时存储用户设置需要caching的数据。 task和所有的executor的生命周期为整个程序的运行过程。

调用一个Spark内部的action会产生一个Spark job来完成它。为确定这些job的实际内容，Spark检查RDD的DAG再计算出执行计划。这个plan以最远端的RDD为起点，产生结果RDD的action为结束。

执行计划由一系列的stage组成，stage是job的transformation的组合，stage对应一系列对于不同的数据集执行相同代码的task。每个stage包含不需要shuffle数据的transformation的序列。

Spark支持需要wide依赖的transformation， 比如groupByKey， reduceByKey。在这种依赖中，计算得到一个partition中的数据需要从父RDD中的多个partition中读取数据。所有拥有相同key的元组最终会被聚合到同一个partition中，被同一个task处理。为了完成这种操作，Sprak需要对数据进行shuffle，意味着数据需要在集群内传递，最终生成由新的partition集合组成的新的stage。

在每个stage的边界，数据被父阶段中的任务写入磁盘，然后通过网络被子阶段的任务获取。因此它们会产生大量的磁盘和网络IO，因此阶段边界可能需要消耗很多资源，应尽可能避免。父阶段的数据分区数可能与子阶段不同，可能触发边界的转换通常接受`numPartitions`参数来决定子阶段数据分区的数量。

调整阶段边界处的分区数通常会决定应用程序的性能。

选择算子布局（arrangement of operators）的主要目的是减少shuffle的次数和数据量。这是因为shuffle是相当消耗资源的操作，必须将数据写入磁盘，然后通过网络传输。`repartition`， `join`，`cogroup`和任何包含`*By` 或者`ByKey`的转换，都会引发shuffle。


## 什么时候不发生shuffle

使用相同的partitioner把数据分partition，Spark可以避免shuffle。如果RDD包含相同个数的partition， join的时候将不会发生额外的shuffle。因为这里的RDD使用相同的hash方式进行partition，所以全部RDD中同一个partition中的key的集合都是相同的。
