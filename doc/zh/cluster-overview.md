---
layout: global
title: 集群模式概述
---

本文档简要介绍了 Spark 如何在群集上运行，以使其更易于理解所涉及的组件。阅读 [应用提交指南](submitting-applications.html)
了解如何在集群上启动应用程序。

# 组件

Spark 应用程序作为独立的进程在集群上运行，由主程序中的 `SparkContext` 对象（称为 _driver program_）来协调。

具体来说，为了在集群上运行，SparkContext 可以连接到多种类型的 _cluster managers_（Spark 自己的 standalone 集群管理器，Mesos 或 YARN），它们跨应用程序分配资源。 一旦连接，Spark 将在集群中的节点上获取 *executors*，这是运行计算并为应用程序存储数据的进程。接下来，它将您的应用程序代码（由 JAR 或 Python 文件传递给 SparkContext）发送到 executors 执行。 最后，SparkContext 将 *tasks* 发送给执行程序运行。

<p style="text-align: center;">
  <img src="img/cluster-overview.png" title="Spark cluster components" alt="Spark cluster components" />
</p>

有关这个架构有几个有用的地方要注意:
1. 每个应用程序都获得自己的 executor 进程，这些进程在整个应用程序的持续时间内保持不变，并在多个线程中运行任务。 这有利于将应用程序彼此隔离，在调度端（每个驱动程序安排其自己的任务）和执行端（来自不同应用程序的任务在不同的JVM中运行）。 但是，这也意味着数据不能在不写入外部存储系统的情况下在不同的Spark应用程序（SparkContext的实例）之间共享。
2. Spark与底层集群管理器无关。 只要可以获取执行程序进程，并且彼此进行通信，即使在一个集群管理器上运行也是比较容易的也支持其它应用程序（例如Mesos/YARN）。
3. 驱动程序必须在其生命周期中监听并接受其执行器的传入连接（例如，请参阅 [在网络配置章节中的 spark.driver.port 部分](configuration.html＃networking)。 因此，驱动程序必须能够从工作节点中网络寻址。
4. 因为驱动程序调度集群上的任务，所以它应该靠近工作节点运行，最好在同一个局域网上运行。 如果您想要远程发送请求到集群，最好是向驱动程序打开一个RPC，并从附近提交操作，而不是在远离工作节点的地方运行驱动程序。

# Cluster Manager（群集管理器）类型

系统目前支持三种集群管理器:
* [Standalone](spark-standalone.html) -- Spark 中包含一个简单的集群管理器，可以轻松设置集群。

* [Apache Mesos](running-on-mesos.html) --  一个通用的 Cluster Manager，它也可以运行 Hadoop MapReduce 和其它服务应用.
* [Hadoop YARN](running-on-yarn.html) --Hadoop 2的资源管理器。
* [Kubernetes (experimental)](https://github.com/apache-spark-on-k8s/spark) -- 除了上述之外，还有 Kubernetes 的实验支持。 Kubernetes 提供以容器为中心的基础设施的开源平台。 Kubernetes 的支持正在 apache-spark-on-k8s Github 组织中积极开发。有关文档，请参阅该项目的README。

# 提交应用程序

应用程序可以使用`spark-submit`脚本提交给任何类型的集群。The [application submission guide](submitting-applications.html)介绍了如何做到这一点。

# 监控

每个 driver 都有一个 Web UI，通常在端口 4040 上，可以显示有关正在运行的任务，执行程序和存储使用情况的信息。 只需在 Web 浏览器中的`http://<driver-node>:4040` 中访问此UI。[监控指南](monitoring.html) 中还介绍了其他监控选项。

# 任务调度

Spark 可以控制不同应用程序（集群管理器级别）和应用程序（如果多个计算发生在同一个 SparkContext 上）的资源分配。 在 [任务调度概述](job-scheduling.html) 中更详细地描述了这一点。

# 术语

下表总结了您将看到的用于引用集群概念的术语：

<table class="table">
  <thead>
    <tr><th style="width: 130px;">Term（术语）</th><th>Meaning（含义）</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>Application</td>
      <td>基于 Spark 的用户程序。 由驱动程序和集群上的执行器组成。</td>
    </tr>
    <tr>
      <td>Application jar</td>
      <td>
        包含用户的 Spark 应用程序的 jar。 在某些情况下，用户将希望创建一个包含其应用程序及其依赖关系的 "uber jar"。 用户的 jar 不应该包含 Hadoop 或 Spark 库，但这些将在运行时添加。
      </td>
    </tr>
    <tr>
      <td>Driver program</td>
      <td>该进程运行应用程序的 main() 函数并创建 SparkContext</td>
    </tr>
    <tr>
      <td>Cluster manager</td>
      <td>用于在集群上获取资源的外部服务(e.g. standalone manager, Mesos, YARN)</td>
    </tr>
    <tr>
      <td>Deploy mode</td>
      <td>区分驱动程序进程的运行位置。 在 "cluster" 模式下，该框架在集群中启动驱动程序。 在 "client" 模式下，提交者在集群外部启动驱动程序。</td>
    </tr>
    <tr>
      <td>Worker node Workr</td>
      <td>可以在群集中运行应用程序代码的任何节点</td>
    </tr>
    <tr>
      <td>Executor</td>
      <td>为工作节点上的应用程序启动的进程，该进程运行任务并将数据保留在内存或磁盘存储器中。 每个应用程序都有自己的执行者。</td>
    </tr>
    <tr>
      <td>Task</td>
      <td>将发送给一个 executor 的任务集合</td>
    </tr>
    <tr>
      <td>Job</td>
      <td>一个由多个任务组成的并行计算，这些任务可以响应Spark action（例如 <code>save</code>, <code>collect</code>）而产生; 你会看到驱动程序日志中使用的这个术语。</td>
    </tr>
    <tr>
      <td>Stage</td>
      <td>每个作业被分成较小的任务称为 <em>stages</em>,这些 tasks 相互依赖（类似于 MapReduce 中的映射并减少阶段）; 您会看到驱动程序日志中使用的这个术语。</td>
    </tr>
  </tbody>
</table>
