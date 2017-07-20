---
layout: global
title: 集群模式概述
---
该文档给出了 Spark 如何在集群上运行、使之更容易来理解所涉及到的组件的简短概述。通过阅读 [应用提交指南](submitting-applications.html)
来学习关于在集群上启动应用。

# 组件


Spark 应用在集群上作为独立的进程组来运行，在您的 main 程序中通过 SparkContext 来协调（称之为 driver 程序）。

具体的说，为了运行在集群上，SparkContext 可以连接至几种类型的 Cluster Manager（既可以用 Spark 自己的 Standlone Cluster Manager，或者 Mesos，也可以使用 YARN），它们会分配应用的资源。一旦连接上，Spark 获得集群中节点上的 Executor，这些进程可以运行计算并且为您的应用存储数据。接下来，它将发送您的应用代码（通过 JAR 或者 Python 文件定义传递给 SparkContext）至 Executor。最终，SparkContext 将发送 Task 到 Executor 以运行。

<p style="text-align: center;">
  <img src="img/cluster-overview.png" title="Spark cluster components" alt="Spark cluster components" />
</p>

这里有几个关于这个架构需要注意的地方 :
1. 每个应用获取到它自己的 Executor 进程，它们会保持在整个应用的生命周期中并且在多个线程中运行 Task（任务）。这样做的优点是把应用互相隔离，在调度方面（每个 driver 调度它自己的 task）和 Executor 方面（来自不同应用的 task 运行在不同的 JVM 中）。然而，这也意味着若是不把数据写到外部的存储系统中的话，数据就不能够被不同的 Spark 应用（SparkContext 的实例）之间共享。
2. Spark 是不知道底层的 Cluster Manager 到底是什么类型的。只要它能够获得 Executor 进程，并且它们可以和彼此之间通信，那么即使是在一个也支持其它应用的 Cluster Manager（例如，Mesos / YARN）上来运行它也是相对简单的。
3. Driver 程序必须在自己的生命周期内（例如，请参阅 [在网络配置章节中的 spark.driver.port 章节](configuration.html＃networking)。 监听和接受来自它的 Executor 的连接请求。同样的，driver 程序必须可以从 worker 节点上网络寻址（就是网络没问题）。
4. 因为 driver 调度了集群上的 task（任务），更好的方式应该是在相同的局域网中靠近 worker 的节点上运行。如果您不喜欢发送请求到远程的集群，倒不如打开一个 RPC 至 driver 并让它就近提交操作而不是从很远的 worker 节点上运行一个 driver。

# Cluster Manager 类型

系统目前支持三种 Cluster Manager:
* [Standalone](spark-standalone.html) -- 包含在 Spark 中使得它更容易来安装集群的一个简单的 Cluster Manager。

* [Apache Mesos](running-on-mesos.html) --  一个通用的 Cluster Manager，它也可以运行 Hadoop MapReduce 和其它服务应用。
* [Hadoop YARN](running-on-yarn.html) --Hadoop 2 中的 resource manager（资源管理器）。
* [Kubernetes (experimental)](https://github.com/apache-spark-on-k8s/spark) -- 除了上述之外，还有 Kubernetes 的实验支持。 Kubernetes 提供以容器为中心的基础设施的开源平台。 Kubernetes 的支持正在 apache-spark-on-k8s Github 组织中积极开发。有关文档，请参阅该项目的 README。

# 提交应用程序

使用 spark-submit 脚本可以提交应用至任何类型的集群。在 [application submission guide](submitting-applications.html) 介绍了如何做到这一点。

# 监控

每个 driver 都有一个 Web UI，通常在端口 4040 上，可以显示有关正在运行的 task，executor，和存储使用情况的信息。 只需在 Web 浏览器中的`http://<driver-node>:4040` 中访问此 UI。[监控指南](monitoring.html) 中还介绍了其他监控选项。

# Job 调度

Spark 即可以在应用间（Cluster Manager 级别），也可以在应用内（如果多个计算发生在相同的 SparkContext 上时）控制资源分配。 在 [任务调度概述](job-scheduling.html) 中更详细地描述了这一点。

# 术语

下表总结了您将看到的用于引用集群概念的术语：

<table class="table">
  <thead>
    <tr><th style="width: 130px;">Term（术语）</th><th>Meaning（含义）</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>Application</td>
      <td>用户构建在 Spark 上的程序。由集群上的一个 driver 程序和多个 executor 组成。</td>
    </tr>
    <tr>
      <td>Application jar</td>
      <td>
       一个包含用户 Spark 应用的 Jar。有时候用户会想要去创建一个包含他们应用以及它的依赖的 “uber jar”。用户的 Jar 应该没有包括 Hadoop 或者 Spark 库，然而，它们将会在运行时被添加。
      </td>
    </tr>
    <tr>
      <td>Driver program</td>
      <td>该进程运行应用的 main() 方法并且创建了 SparkContext。</td>
    </tr>
    <tr>
      <td>Cluster manager</td>
      <td>一个外部的用于获取集群上资源的服务。（例如，Standlone Manager，Mesos，YARN）</td>
    </tr>
    <tr>
      <td>Deploy mode</td>
      <td>根据 driver 程序运行的地方区别。在 “Cluster” 模式中，框架在群集内部启动 driver。在 “Client” 模式中，submitter（提交者）在 Custer 外部启动 driver。</td>
    </tr>
    <tr>
      <td>Worker node</td>
      <td>任何在集群中可以运行应用代码的节点。</td>
    </tr>
    <tr>
      <td>Executor</td>
      <td>一个为了在 worker 节点上的应用而启动的进程，它运行 task 并且将数据保持在内存中或者硬盘存储。每个应用有它自己的 Executor。</td>
    </tr>
    <tr>
      <td>Task</td>
      <td>一个将要被发送到 Executor 中的工作单元。</td>
    </tr>
    <tr>
      <td>Job</td>
      <td>一个由多个任务组成的并行计算，并且能从 Spark action 中获取响应（例如 <code>save</code>, <code>collect</code>）; 您将在 driver 的日志中看到这个术语。</td>
    </tr>
    <tr>
      <td>Stage</td>
      <td>每个 Job 被拆分成更小的被称作 stage（阶段） 的 task（任务） 组，stage 彼此之间是相互依赖的（与 MapReduce 中的 map 和 reduce stage 相似）。您将在 driver 的日志中看到这个术语。</td>
    </tr>
  </tbody>
</table>
