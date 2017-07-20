---
layout: global
title: Cluster Mode Overview
---

This document gives a short overview of how Spark runs on clusters, to make it easier to understand
the components involved. Read through the [application submission guide](submitting-applications.html)
to learn about launching applications on a cluster.

该文档给出了 Spark 如何在集群上运行、使之更容易来理解所涉及到的组件的简短概述。通过阅读 应用提交指南 来学习关于在集群上启动应用。

# Components
组件

Spark applications run as independent sets of processes on a cluster, coordinated by the `SparkContext`
object in your main program (called the _driver program_).

Spark应用程序作为独立的进程在集群上运行，由主程序中的“SparkContext”对象（称为driver program）来协调。



Specifically, to run on a cluster, the SparkContext can connect to several types of _cluster managers_
(either Spark's own standalone cluster manager, Mesos or YARN), which allocate resources across
applications. Once connected, Spark acquires *executors* on nodes in the cluster, which are
processes that run computations and store data for your application.
Next, it sends your application code (defined by JAR or Python files passed to SparkContext) to
the executors. Finally, SparkContext sends *tasks* to the executors to run.

具体的说，为了运行在集群上，SparkContext 可以连接至几种类型的 Cluster Manager（既可以用 Spark 自己的 Standlone Cluster Manager，或者 Mesos，也可以使用 YARN），它们会分配应用的资源。一旦连接上，Spark 获得集群中节点上的 Executor，这些进程可以运行计算并且为您的应用存储数据。接下来，它将发送您的应用代码（通过 JAR 或者 Python 文件定义传递给 SparkContext）至 Executor。最终，SparkContext 将发送 Task 到 Executor 以运行。

<p style="text-align: center;">
  <img src="img/cluster-overview.png" title="Spark cluster components" alt="Spark cluster components" />
</p>

There are several useful things to note about this architecture:

这里有几个关于这个架构需要注意的地方 :
1. Each application gets its own executor processes, which stay up for the duration of the whole
   application and run tasks in multiple threads. This has the benefit of isolating applications
   from each other, on both the scheduling side (each driver schedules its own tasks) and executor
   side (tasks from different applications run in different JVMs). However, it also means that
   data cannot be shared across different Spark applications (instances of SparkContext) without
   writing it to an external storage system.

   1.每个应用获取到它自己的 Executor 进程，它们会保持在整个应用的生命周期中并且在多个线程中运行 Task（任务）。这样做的优点是把应用互相隔离，在调度方面（每个 driver 调度它自己的 task）和 Executor 方面（来自不同应用的 task 运行在不同的 JVM 中）。然而，这也意味着若是不把数据写到外部的存储系统中的话，数据就不能够被不同的 Spark 应用（SparkContext 的实例）之间共享。
2. Spark is agnostic to the underlying cluster manager. As long as it can acquire executor
   processes, and these communicate with each other, it is relatively easy to run it even on a
   cluster manager that also supports other applications (e.g. Mesos/YARN).

   2.Spark 是不知道底层的 Cluster Manager 到底是什么类型的。只要它能够获得 Executor 进程，并且它们可以和彼此之间通信，那么即使是在一个也支持其它应用的 Cluster Manager（例如，Mesos / YARN）上来运行它也是相对简单的。
3. The driver program must listen for and accept incoming connections from its executors throughout
   its lifetime (e.g., see [spark.driver.port in the network config
   section](configuration.html#networking)). As such, the driver program must be network
   addressable from the worker nodes.

   3.Driver 程序必须在自己的生命周期内（例如，请看 在网络中配置 spark.driver.port 章节）监听和接受来自它的 Executor 的连接请求。同样的，driver 程序必须可以从 worker 节点上网络寻址（就是网络没问题）。
4. Because the driver schedules tasks on the cluster, it should be run close to the worker
   nodes, preferably on the same local area network. If you'd like to send requests to the
   cluster remotely, it's better to open an RPC to the driver and have it submit operations
   from nearby than to run a driver far away from the worker nodes.

  4，因为 driver 调度了集群上的 task（任务），更好的方式应该是在相同的局域网中靠近 worker 的节点上运行。如果您不喜欢发送请求到远程的集群，倒不如打开一个 RPC 至 driver 并让它就近提交操作而不是从很远的 worker 节点上运行一个 driver。
# Cluster Manager Types
Cluster Manager 类型

The system currently supports three cluster managers:

系统目前支持三种Cluster Manager：

* [Standalone](spark-standalone.html) -- a simple cluster manager included with Spark that makes it
  easy to set up a cluster.
 
* [Standalone](spark-standalone.html) -- 包含在 Spark 中使得它更容易来安装集群的一个简单的 Cluster Manager

* [Apache Mesos](running-on-mesos.html) -- a general cluster manager that can also run Hadoop MapReduce
  and service applications.

 * [Apache Mesos](running-on-mesos.html) --  一个通用的 Cluster Manager，它也可以运行 Hadoop MapReduce 和其它服务应用。
* [Hadoop YARN](running-on-yarn.html) -- the resource manager in Hadoop 2.
* [Hadoop YARN](running-on-yarn.html) --Hadoop 2 中的 resource manager（资源管理器）。
* [Kubernetes (experimental)](https://github.com/apache-spark-on-k8s/spark) -- In addition to the above,
there is experimental support for Kubernetes. Kubernetes is an open-source platform
for providing container-centric infrastructure. Kubernetes support is being actively
developed in an [apache-spark-on-k8s](https://github.com/apache-spark-on-k8s/) Github organization. 
For documentation, refer to that project's README.
* [Kubernetes (experimental)](https://github.com/apache-spark-on-k8s/spark) --除了上述之外，还有 Kubernetes 的实验支持。Kubernetes 提供以容器为中心的基础设施的开源平台。Kubernetes 的支持正在[apache-spark-on-k8s](https://github.com/apache-spark-on-k8s/) Github 组织中积极开发。 有关文档，请参阅该项目的 README。

# Submitting Applications
提交应用程序

Applications can be submitted to a cluster of any type using the `spark-submit` script.
The [application submission guide](submitting-applications.html) describes how to do this.

应用程序可以使用 `spark-submit` 脚本提交给任何类型的集群。在 [ 应用提交指南](submitting-applications.html)介绍了如何做到这一点。

# Monitoring
监控

Each driver program has a web UI, typically on port 4040, that displays information about running
tasks, executors, and storage usage. Simply go to `http://<driver-node>:4040` in a web browser to
access this UI. The [monitoring guide](monitoring.html) also describes other monitoring options.

每个驱动程序都有一个Web UI，通常在端口4040上，可以显示有关正在运行的 task，executor，和存储使用情况的信息。只需在 Web 浏览器中的 `http：// <driver-node>：4040`中访问此 UI。[监控指南](monitoring.html)中还介绍了其他监控选项。

# Job Scheduling
Job调度

Spark gives control over resource allocation both _across_ applications (at the level of the cluster
manager) and _within_ applications (if multiple computations are happening on the same SparkContext).
The [job scheduling overview](job-scheduling.html) describes this in more detail.

Spark 即可以在应用间（Cluster Manager 级别），也可以在应用内（如果多个计算发生在相同的 SparkContext 上时）控制资源分配。[Job 调度指南](job-scheduling.html) 描述的更详细。

# Glossary
术语

The following table summarizes terms you'll see used to refer to cluster concepts:

下表中总结了您将会看到用于涉及到集群时的术语：

<table class="table">
  <thead>
    <tr><th style="width: 130px;">Term</th><th>Meaning</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>Application</td>
      <td>User program built on Spark. Consists of a <em>driver program</em> and <em>executors</em> on the cluster.用户构建在 Spark 上的程序。由集群上的一个 driver 程序和多个 executor 组成。</td>
    </tr>
    <tr>
      <td>Application jar</td>
      <td>
        A jar containing the user's Spark application. In some cases users will want to create
        an "uber jar" containing their application along with its dependencies. The user's jar
        should never include Hadoop or Spark libraries, however, these will be added at runtime.一个包含用户 Spark 应用的 Jar。有时候用户会想要去创建一个包含他们应用以及它的依赖的 “uber jar”。用户的 Jar 应该没有包括 Hadoop 或者 Spark 库，然而，它们将会在运行时被添加。
      </td>
    </tr>
    <tr>
      <td>Driver program</td>
      <td>The process running the main() function of the application and creating the SparkContext该进程运行应用程序的 main() 函数并创SparkContext</td>
    </tr>
    <tr>
      <td>Cluster manager</td>
      <td>An external service for acquiring resources on the cluster (e.g. standalone manager, Mesos, YARN)一个外部的用于获取集群上资源的服务。（例如，Standlone Manager，Mesos，YARN）</td>
    </tr>
    <tr>
      <td>Deploy mode</td>
      <td>Distinguishes where the driver process runs. In "cluster" mode, the framework launches
        the driver inside of the cluster. In "client" mode, the submitter launches the driver
        outside of the cluster.根据 driver 程序运行的地方区别。在 “Cluster” 模式中，框架在群集内部启动 driver。在 “Client” 模式中，submitter（提交者）在 Custer 外部启动 driver。</td>
    </tr>
    <tr>
      <td>Worker node</td>
      <td>Any node that can run application code in the cluster任何在集群中可以运行应用代码的节点。</td>
    </tr>
    <tr>
      <td>Executor</td>
      <td>A process launched for an application on a worker node, that runs tasks and keeps data in memory
        or disk storage across them. Each application has its own executors.一个为了在 worker 节点上的应用而启动的进程，它运行 task 并且将数据保持在内存中或者硬盘存储。每个应用有它自己的 Executor。</td>
    </tr>
    <tr>
      <td>Task</td>
      <td>A unit of work that will be sent to one executor一个将要被发送到 Executor 中的工作单元。</td>
    </tr>
    <tr>
      <td>Job</td>
      <td>A parallel computation consisting of multiple tasks that gets spawned in response to a Spark action
        (e.g. <code>save</code>, <code>collect</code>); you'll see this term used in the driver's logs.一个由多个 task 组成的并行计算，并且能从 Spark action 中获取响应（例如，save，collect），您将在 driver 的日志中看到这个术语。</td>
    </tr>
    <tr>
      <td>Stage</td>
      <td>Each job gets divided into smaller sets of tasks called <em>stages</em> that depend on each other
        (similar to the map and reduce stages in MapReduce); you'll see this term used in the driver's logs.每个 Job 被拆分成更小的被称作 stage（阶段） 的 task（任务） 组，stage 彼此之间是相互依赖的（与 MapReduce 中的 map 和 reduce stage 相似）。您将在 driver 的日志中看到这个术语。</td>
    </tr>
  </tbody>
</table>
