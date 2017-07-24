---
layout: global
title: 硬件配置
---

Spark 开发者都会遇到一个常见问题，那就是如何为 Spark 配置硬件。然而正确的硬件配置取决于使用的场景，我们提出以下建议。

# 存储系统

因为大多数 Spark 作业都很可能必须从外部存储系统（例如 Hadoop 文件系统或者 HBase）读取输入的数据，所以部署 Spark 时**尽可能靠近这些系统**是很重要的。我们建议如下 : 

* 如果可以，在 HDFS 相同的节点上运行 Spark 。最简单的方法是在相同节点上设置 Spark [独立集群模式](spark-standalone.html)，并且配置 Spark 和 Hadoop 的内存和 CPU 的使用以避免干扰（Hadoop 的相关选项为 : 设置每个任务内存大小的选项 `mapred.child.java.opts` 以及设置任务数量的选项 `mapred.tasktracker.map.tasks.maximum` 和 `mapred.tasktracker.reduce.tasks.maximum`）。当然您也可以在常用的集群管理器（比如 [Mesos](running-on-mesos.html) 或者 [YARN](running-on-yarn.html)）上运行  Hadoop 和 Spark。

* 如果不可以在相同的节点上运行，建议在与 HDFS 相同的局域网中的不同节点上运行 Spark 。

* 对于低延迟数据存储（如HBase），在这些存储系统不同的节点上运行计算作业来可能更有利于避免干扰。

# 本地磁盘

虽然 Spark 可以在内存中执行大量计算，但是他仍然使用本地磁盘来存储不适合内存存储的数据以及各个阶段的中间结果。我们建议每个节点配置 **4-8** 个磁盘，并且不使用 RAID 配置(只作为单独挂载点)。在 Linux 中,使用 [noatime选项](http://www.centos.org/docs/5/html/Global_File_System/s2-manage-mountnoatime.html) 挂载磁盘以减少不必要的写入。在 Spark 中，[配置](configuration.html) `spark.local.dir` 变量为逗号分隔的本地磁盘列表。如果您正在运行 HDFS，可以使用与 HDFS 相同的磁盘。

# 内存

一般来说，Spark 可以在每台机器 **8GB 到数百 GB** 内存的任何地方正常运行。在所有情况下，我们建议只为 Spark 分配最多75% 的内存；其余部分供操作系统和缓存区高速缓存存储器使用。

您需要多少内存取决于您的应用程序。如果您需要确定的应用程序中某个特定数据集占用内存的大小，您可以把这个数据集加载到一个 Spark RDD 中，然后在 Spark 监控 UI 页面（`http://<driver-node>:4040`）中的 Storage 选项卡下查看它在内存中的大小。需要注意的是，存储级别和序列化格式对内存使用量有很大的影响 - 如何减少内存使用量的建议，请参阅[调优指南](tuning.html)。

最后,需要注意的是 Java 虚拟机在超过 200GB 的 RAM 时表现得并不好。如果您购置的机器有比这更多的 RAM ，您可以在每个节点上运行多个 Worker 的 JVM 实例。在 Spark 的 [standalone mode](spark-standalone.html) 下,您可以通过 `conf/spark-env.sh` 中的 `SPARK_WORKER_INSTANCES` 和 `SPARK_WORKER_CORES` 两个参数来分别设置每个节点的 Worker 数量和每个 Worker 使用的 Core 数量。
# 网络

根据我们的经验，当数据在内存中时，很多 Spark 应用程序跟网络有密切的关系。使用 **10 千兆位**以太网或者更快的网络是让这些应用程序变快的最佳方式。这对于 “distributed reduce” 类的应用程序来说尤其如此，例如 group-by 、reduce-by 和 SQL join。任何程序都可以在应用程序监控 UI 页面 (`http://<driver-node>:4040`) 中查看 Spark 通过网络传输的数据量。

# CPU Cores

因为 Spark 实行线程之间的最小共享，所以 Spark 可以很好地在每台机器上扩展数十个 CPU Core。您应该为每台机器至少配置 **8-16 个 Core**。根据您工作负载的 CPU 成本，您可能还需要更多：当数据都在内存中时，大多数应用程序就只跟 CPU 或者网络有关了。
