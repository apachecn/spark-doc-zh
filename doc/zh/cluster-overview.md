---
layout: global
title: Cluster Mode Overview
---

This document gives a short overview of how Spark runs on clusters, to make it easier to understand
the components involved. Read through the [application submission guide](submitting-applications.html)
to learn about launching applications on a cluster.

本文档简要介绍了Spark如何在群集上运行，以使其更易于理解所涉及的组件。阅读[应用提交指南]（submitting-applications.html）
了解如何在集群上启动应用程序。

# Components
组件

Spark applications run as independent sets of processes on a cluster, coordinated by the `SparkContext`
object in your main program (called the _driver program_).

Spark应用程序作为独立的进程在集群上运行，由主程序中的“SparkContext”对象（称为_driver program_）来协调。



Specifically, to run on a cluster, the SparkContext can connect to several types of _cluster managers_
(either Spark's own standalone cluster manager, Mesos or YARN), which allocate resources across
applications. Once connected, Spark acquires *executors* on nodes in the cluster, which are
processes that run computations and store data for your application.
Next, it sends your application code (defined by JAR or Python files passed to SparkContext) to
the executors. Finally, SparkContext sends *tasks* to the executors to run.

具体来说，为了在集群上运行，SparkContext可以连接到多种类型的 _cluster managers_（Spark自己的standalone集群管理器，Mesos或YARN），它们跨应用程序分配资源。 一旦连接，Spark将在集群中的节点上获取*executors*，这是运行计算并为应用程序存储数据的进程。接下来，它将您的应用程序代码（由JAR或Python文件传递给SparkContext）发送到executors执行。 最后，SparkContext将*tasks*发送给执行程序运行。

<p style="text-align: center;">
  <img src="img/cluster-overview.png" title="Spark cluster components" alt="Spark cluster components" />
</p>

There are several useful things to note about this architecture:

有关这个架构有几个有用的事情要注意：
1. Each application gets its own executor processes, which stay up for the duration of the whole
   application and run tasks in multiple threads. This has the benefit of isolating applications
   from each other, on both the scheduling side (each driver schedules its own tasks) and executor
   side (tasks from different applications run in different JVMs). However, it also means that
   data cannot be shared across different Spark applications (instances of SparkContext) without
   writing it to an external storage system.

   1.每个应用程序都获得自己的executor进程，这些进程在整个应用程序的持续时间内保持不变，并在多个线程中运行任务。 这有利于将应用程序彼此隔离，在调度端（每个驱动程序安排其自己的任务）和执行端（来自不同应用程序的任务在不同的JVM中运行）。 但是，这也意味着数据不能在不写入外部存储系统的情况下在不同的Spark应用程序（SparkContext的实例）之间共享。
2. Spark is agnostic to the underlying cluster manager. As long as it can acquire executor
   processes, and these communicate with each other, it is relatively easy to run it even on a
   cluster manager that also supports other applications (e.g. Mesos/YARN).

   2.Spark与底层集群管理器无关。 只要可以获取执行程序进程，并且彼此进行通信，即使在一个集群管理器上运行也是比较容易的也支持其他应用程序（例如Mesos / YARN）。
3. The driver program must listen for and accept incoming connections from its executors throughout
   its lifetime (e.g., see [spark.driver.port in the network config
   section](configuration.html#networking)). As such, the driver program must be network
   addressable from the worker nodes.

   3.驱动程序必须在其生命周期中监听并接受其执行器的传入连接（例如，请参阅[spark.driver.port在网络配置章节]（configuration.html＃networking））。 因此，驱动程序必须能够从工作节点中网络寻址。
4. Because the driver schedules tasks on the cluster, it should be run close to the worker
   nodes, preferably on the same local area network. If you'd like to send requests to the
   cluster remotely, it's better to open an RPC to the driver and have it submit operations
   from nearby than to run a driver far away from the worker nodes.

4，因为驱动程序调度集群上的任务，所以它应该靠近工作节点运行，最好在同一个局域网上运行。 如果您想要远程发送请求到集群，最好是向驱动程序打开一个RPC，并从附近提交操作，而不是在远离工作节点的地方运行驱动程序。
# Cluster Manager Types
群集管理器类型

The system currently supports three cluster managers:

系统目前支持三种集群管理器：

* [Standalone](spark-standalone.html) -- a simple cluster manager included with Spark that makes it
  easy to set up a cluster.
 
* [Standalone](spark-standalone.html) -- Spark中包含一个简单的集群管理器，可以轻松设置集群。

* [Apache Mesos](running-on-mesos.html) -- a general cluster manager that can also run Hadoop MapReduce
  and service applications.

 * [Apache Mesos](running-on-mesos.html) --  也可以运行Hadoop MapReduce和服务应用程序的通用集群管理器。
* [Hadoop YARN](running-on-yarn.html) -- the resource manager in Hadoop 2.
* [Hadoop YARN](running-on-yarn.html) --Hadoop 2的资源管理器。
* [Kubernetes (experimental)](https://github.com/apache-spark-on-k8s/spark) -- In addition to the above,
there is experimental support for Kubernetes. Kubernetes is an open-source platform
for providing container-centric infrastructure. Kubernetes support is being actively
developed in an [apache-spark-on-k8s](https://github.com/apache-spark-on-k8s/) Github organization. 
For documentation, refer to that project's README.
* [Kubernetes (experimental)](https://github.com/apache-spark-on-k8s/spark) --除了上述之外，还有Kubernetes的实验支持。 Kubernetes提供以容器为中心的基础设施的开源平台。 Kubernetes的支持正在apache-spark-on-k8s Github组织中积极开发。 有关文档，请参阅该项目的README。

# Submitting Applications
提交应用程序

Applications can be submitted to a cluster of any type using the `spark-submit` script.
The [application submission guide](submitting-applications.html) describes how to do this.

应用程序可以使用`spark-submit`脚本提交给任何类型的集群。The [application submission guide](submitting-applications.html)介绍了如何做到这一点。

# Monitoring
监控

Each driver program has a web UI, typically on port 4040, that displays information about running
tasks, executors, and storage usage. Simply go to `http://<driver-node>:4040` in a web browser to
access this UI. The [monitoring guide](monitoring.html) also describes other monitoring options.

每个驱动程序都有一个Web UI，通常在端口4040上，可以显示有关正在运行的任务，执行程序和存储使用情况的信息。 只需在Web浏览器中的`http：// <driver-node>：4040`中访问此UI。[monitoring guide](monitoring.html)中还介绍了其他监控选项。

# Job Scheduling
任务调度

Spark gives control over resource allocation both _across_ applications (at the level of the cluster
manager) and _within_ applications (if multiple computations are happening on the same SparkContext).
The [job scheduling overview](job-scheduling.html) describes this in more detail.

Spark可以控制不同应用程序（集群管理器级别）和应用程序（如果多个计算发生在同一个SparkContext上）的资源分配。 The [job scheduling overview](job-scheduling.html)更详细地描述了这一点。

# Glossary
术语

The following table summarizes terms you'll see used to refer to cluster concepts:

下表总结了您将看到的用于引用集群概念的术语：

<table class="table">
  <thead>
    <tr><th style="width: 130px;">Term</th><th>Meaning</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>Application应用程序</td>
      <td>User program built on Spark. Consists of a <em>driver program</em> and <em>executors</em> on the cluster.基于Spark的用户程序。 由驱动程序和集群上的执行器组成。</td>
    </tr>
    <tr>
      <td>Application jar应用程序jar</td>
      <td>
        A jar containing the user's Spark application. In some cases users will want to create
        an "uber jar" containing their application along with its dependencies. The user's jar
        should never include Hadoop or Spark libraries, however, these will be added at runtime.包含用户的Spark应用程序的jar。 在某些情况下，用户将希望创建一个包含其应用程序及其依赖关系的“uber jar”。 用户的jar不应该包含Hadoop或Spark库，但这些将在运行时添加。
      </td>
    </tr>
    <tr>
      <td>Driver program驱动程序</td>
      <td>The process running the main() function of the application and creating the SparkContext该进程运行应用程序的main()函数并创建SparkContext</td>
    </tr>
    <tr>
      <td>Cluster manager集群管理器</td>
      <td>An external service for acquiring resources on the cluster (e.g. standalone manager, Mesos, YARN)用于在集群上获取资源的外部服务(e.g. standalone manager, Mesos, YARN)</td>
    </tr>
    <tr>
      <td>Deploy mode部署模式</td>
      <td>Distinguishes where the driver process runs. In "cluster" mode, the framework launches
        the driver inside of the cluster. In "client" mode, the submitter launches the driver
        outside of the cluster.区分驱动程序进程的运行位置。 在“集群”模式下，该框架在集群中启动驱动程序。 在“客户端”模式下，提交者在集群外部启动驱动程序。</td>
    </tr>
    <tr>
      <td>Worker node Workr节点</td>
      <td>Any node that can run application code in the cluster可以在群集中运行应用程序代码的任何节点</td>
    </tr>
    <tr>
      <td>Executor 执行者</td>
      <td>A process launched for an application on a worker node, that runs tasks and keeps data in memory
        or disk storage across them. Each application has its own executors.为工作节点上的应用程序启动的进程，该进程运行任务并将数据保留在内存或磁盘存储器中。 每个应用程序都有自己的执行者。</td>
    </tr>
    <tr>
      <td>Task任务</td>
      <td>A unit of work that will be sent to one executor将发送给一个executor的任务集合</td>
    </tr>
    <tr>
      <td>Job</td>
      <td>A parallel computation consisting of multiple tasks that gets spawned in response to a Spark action
        (e.g. <code>save</code>, <code>collect</code>); you'll see this term used in the driver's logs.一个由多个任务组成的并行计算，这些任务可以响应Spark action（例如save，collect）而产生; 你会看到驱动程序日志中使用的这个术语。</td>
    </tr>
    <tr>
      <td>Stage</td>
      <td>Each job gets divided into smaller sets of tasks called <em>stages</em> that depend on each other
        (similar to the map and reduce stages in MapReduce); you'll see this term used in the driver's logs.每个作业被分成较小的任务称为stage,这些tasks相互依赖（类似于MapReduce中的映射并减少阶段）; 您会看到驱动程序日志中使用的这个术语。</td>
    </tr>
  </tbody>
</table>
