---
layout: global
displayTitle: Spark 概述
title: 概述
description: Apache Spark SPARK_VERSION_SHORT 官方文档中文版首页
---

Apache Spark 是一个快速的, 多用途的集群计算系统。
它在 Java, Scala, Python 和 R 语言以及一个支持常见的图计算的经过优化的引擎中提供了高级 API。
它还支持一组丰富的高级工具, 包括用于 SQL 和结构化数据处理的 [Spark SQL](sql-programming-guide.html), 用于机器学习的 [MLlib](ml-guide.html), 用于图形处理的 [GraphX](graphx-programming-guide.html), 以及 [Spark Streaming](streaming-programming-guide.html)。

# 下载

从该项目官网的 [下载页面](http://spark.apache.org/downloads.html) 获取 Spark. 该文档用于 Spark {{site.SPARK_VERSION}} 版本. Spark 使用了针对 HDFS 和 YARN 的 Hadoop 的 client libraries（客户端库）. 为了适用于主流的 Hadoop 版本可以下载先前的 package.
用户还可以下载 "Hadoop free" binary, 并且可以 [通过增加 Spark 的 classpath](hadoop-provided.html) Spark 来与任何的 Hadoop 版本一起运行 Spark.
Scala 和 Java 用户可以在他们的工程中使用它的 Maven 坐标来包含 Spark, 并且在将来 Python 用户也可以从 PyPI 中安装 Spark。


如果您希望从源码中构建 Spark, 请访问 [构建 Spark](building-spark.html).


Spark 既可以在 Windows 上又可以在类似 UNIX 的系统（例如, Linux, Mac OS）上运行。它很容易在一台机器上本地运行 - 您只需要在系统 `PATH` 上安装 `Java`, 或者将 `JAVA_HOME` 环境变量指向一个 `Java` 安装目录即可。

Spark 可运行在 Java 8+, Python 2.7+/3.4+ 和 R 3.1+ 的环境上。针对 Scala API, Spark {{site.SPARK_VERSION}}
使用了 Scala {{site.SCALA_BINARY_VERSION}}. 您将需要去使用一个可兼容的 Scala 版本
({{site.SCALA_BINARY_VERSION}}.x).

请注意, 从 Spark 2.2.0 起, 对 Java 7, Python 2.6 和旧的 Hadoop 2.6.5 之前版本的支持均已被删除.

请注意, Scala 2.10 的支持已经不再适用于 Spark 2.1.0, 可能会在 Spark 2.3.0 中删除。

# 运行示例和 Shell

Spark 自带了几个示例程序.  Scala, Java, Python 和 R 示例在
`examples/src/main` 目录中. 要运行 Java 或 Scala 中的某个示例程序, 在最顶层的 Spark 目录中使用
`bin/run-example <class> [params]` 命令即可.（在幕后, 它调用了 [`spark-submit` 脚本](submitting-applications.html)以启动应用程序）。例如,

    ./bin/run-example SparkPi 10

您也可以通过一个改进版的 Scala shell 来运行交互式的 Spark。这是一个来学习该框架比较好的方式。

    ./bin/spark-shell --master local[2]

该 `--master`选项可以指定为为
[针对分布式集群的 master URL](submitting-applications.html#master-urls), 或者 `local` 以使用 1 个线程在本地运行, 或者 `local[N]` 以使用 N 个线程在本地运行。您应该通过使用
`local` 来启动以便测试. 该选项的完整列表, 请使用 `--help` 选项来运行 Spark shell。

Spark 同样支持 Python API。在 Python interpreter（解释器）中运行交互式的 Spark, 请使用
`bin/pyspark`:

    ./bin/pyspark --master local[2]

Python 中也提供了应用示例。例如, 

    ./bin/spark-submit examples/src/main/python/pi.py 10

从 1.4 开始（仅包含了 DataFrames APIs）Spark 也提供了一个用于实验性的 [R API](sparkr.html)。
为了在 R interpreter（解释器）中运行交互式的 Spark, 请执行 `bin/sparkR`:

    ./bin/sparkR --master local[2]

R 中也提供了应用示例。例如, 

    ./bin/spark-submit examples/src/main/r/dataframe.R

# 在集群上运行

该 Spark [集群模式概述](cluster-overview.html) 说明了在集群上运行的主要的概念。
Spark 既可以独立运行, 也可以在一些现有的 Cluster Manager（集群管理器）上运行。它当前提供了几种用于部署的选项: 

* [Standalone Deploy Mode](spark-standalone.html): 在私有集群上部署 Spark 最简单的方式
* [Apache Mesos](running-on-mesos.html)
* [Hadoop YARN](running-on-yarn.html)

# 快速跳转

**编程指南:**

* [快速入门](quick-start.html): 简单的介绍 Spark API; 从这里开始！
* [Spark 编程指南](programming-guide.html): 在 Spark 支持的所有语言（Scala, Java, Python, R）中的详细概述。
* 构建在 Spark 之上的模块:
  * [Spark Streaming](streaming-programming-guide.html): 实时数据流处理
  * [Spark SQL, Datasets, and DataFrames](sql-programming-guide.html): 支持结构化数据和关系查询
  * [MLlib](ml-guide.html): 内置的机器学习库
  * [GraphX](graphx-programming-guide.html): 新一代用于图形处理的 Spark API。

**API 文档:**

* [Spark Scala API (Scaladoc)](http://spark.apache.org/docs/2.2.0/api/scala/index.html#org.apache.spark.package)
* [Spark Java API (Javadoc)](http://spark.apache.org/docs/2.2.0/api/java/index.html)
* [Spark Python API (Sphinx)](http://spark.apache.org/docs/2.2.0/api/python/index.html)
* [Spark R API (Roxygen2)](http://spark.apache.org/docs/2.2.0/api/R/index.html)

**部署指南:**

* [集群概述](cluster-overview.html): 在集群上运行时概念和组件的概述。
* [提交应用](submitting-applications.html): 打包和部署应用
* 部署模式:
  * [Amazon EC2](https://github.com/amplab/spark-ec2):  花费大约5分钟的时间让您在EC2上启动一个集群的脚本
  * [Standalone Deploy Mode](spark-standalone.html): 在不依赖第三方 Cluster Manager 的情况下快速的启动一个独立的集群
  * [Mesos](running-on-mesos.html): 使用 [Apache Mesos](http://mesos.apache.org) 来部署一个私有的集群
  * [YARN](running-on-yarn.html): 在 Hadoop NextGen（YARN）上部署 Spark
  * [Kubernetes (experimental)](https://github.com/apache-spark-on-k8s/spark): 在 Kubernetes 之上部署 Spark

**其它文档:**

* [配置](configuration.html): 通过它的配置系统定制 Spark
* [监控](monitoring.html): 跟踪应用的行为
* [优化指南](tuning.html): 性能优化和内存调优的最佳实践
* [任务调度](job-scheduling.html): 资源调度和任务调度
* [安全性](security.html): Spark 安全性支持
* [硬件挑选](hardware-provisioning.html): 集群硬件挑选的建议
* 与其他存储系统的集成:
  * [OpenStack Swift](storage-openstack-swift.html)
* [构建 Spark](building-spark.html): 使用 Maven 来构建 Spark
* [给 Spark 贡献](http://spark.apache.org/contributing.html)
* [第三方项目](http://spark.apache.org/third-party-projects.html): 其它第三方 Spark 项目的支持

**外部资源:**

* [Spark 首页](http://spark.apache.org)
* [Spark 社区](http://spark.apache.org/community.html) 资源, 包括当地的聚会
* [StackOverflow tag `apache-spark`](http://stackoverflow.com/questions/tagged/apache-spark)
* [Mailing Lists](http://spark.apache.org/mailing-lists.html): 在这里询问关于 Spark 的问题
* [AMP Camps](http://ampcamp.berkeley.edu/): 在 UC Berkeley（加州大学伯克利分校）的一系列的训练营中, 它们的特色是讨论和针对关于 Spark, Spark Streaming, Mesos 的练习, 等等。在这里可以免费获取[视频](http://ampcamp.berkeley.edu/6/),
  [幻灯片](http://ampcamp.berkeley.edu/6/) 和 [练习题](http://ampcamp.berkeley.edu/6/exercises/)。
* [Code Examples](http://spark.apache.org/examples.html): 更多`示例`可以在 Spark 的子文件夹中获取 ([Scala]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/scala/org/apache/spark/examples),
 [Java]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/java/org/apache/spark/examples),
 [Python]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/python),
 [R]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/r))
