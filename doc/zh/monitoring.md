---
layout: global
title: Monitoring and Instrumentation
description: Monitoring, metrics, and instrumentation guide for Spark SPARK_VERSION_SHORT
---

有几种方法来监视Spark应用程序：Web UI，metrics和外部工具。

# Web 界面

每个SparkContext都会启动一个Web UI，默认端口为4040，显示有关应用程序的有用信息。这包括：

* 调度器阶段和任务的列表
* RDD 大小和内存使用的概要信息
* 环境信息
* 正在运行的执行器的信息

您可以通过在Web浏览器中打开 `http://<driver-node>:4040` 来访问此界面。
如果多个SparkContexts在同一主机上运行，则它们将绑定到连续的端口从4040（4041，4042等）开始。

请注意，默认情况下此信息仅适用于运行中的应用程序。要在事后还能通过Web UI查看，请在应用程序启动之前，将`spark.eventLog.enabled`设置为true。 
这配置Spark持久存储以记录Spark事件，再通过编码该信息在UI中进行显示。

## 事后查看

仍然可以通过Spark的历史服务器构建应用程序的UI，
只要应用程序的事件日志存在。
您可以通过执行以下命令启动历史服务器：

    ./sbin/start-history-server.sh

默认情况下，会在 `http://<server-url>:18080` 创建一个Web 界面 ，显示未完成、完成以及其他尝试的任务信息。

当使用file-system提供程序类（见下面 `spark.history.provider`）时，基本日志记录目录必须在`spark.history.fs.logDirectory`配置选项中提供，并且应包含每个代表应用程序的事件日志的子目录。

Spark任务本身必须配置启用记录事件，并将其记录到相同共享的可写目录下。 
例如，如果服务器配置了日志目录`hdfs://namenode/shared/spark-logs`，那么客户端选项将是：

    spark.eventLog.enabled true
    spark.eventLog.dir hdfs://namenode/shared/spark-logs

history server 可以配置如下：

### 环境变量

<table class="table">
  <tr><th style="width:21%">环境变量</th><th>含义</th></tr>
  <tr>
    <td><code>SPARK_DAEMON_MEMORY</code></td>
    <td>history server 内存分配（默认值：1g）</td>
  </tr>
  <tr>
    <td><code>SPARK_DAEMON_JAVA_OPTS</code></td>
    <td>history server JVM选项（默认值：无）</td>
  </tr>
  <tr>
    <td><code>SPARK_PUBLIC_DNS</code></td>
    <td>
      history server 公共地址。如果没有设置，应用程序历史记录的链接可能会使用服务器的内部地址，导致链接断开（默认值：无）。
    </td>
  </tr>
  <tr>
    <td><code>SPARK_HISTORY_OPTS</code></td>
    <td>
      <code>spark.history.*</code> history server 配置选项（默认值：无）
    </td>
  </tr>
</table>

### Spark配置选项

<table class="table">
  <tr><th>属性名称</th><th>默认</th><th>含义</th></tr>
  <tr>
    <td>spark.history.provider</td>
    <td><code>org.apache.spark.deploy.history.FsHistoryProvider</code></td>
    <td>执行应用程序历史后端的类的名称。 目前只有一个实现，由Spark提供，它查找存储在文件系统中的应用程序日志。</td>
  </tr>
  <tr>
    <td>spark.history.fs.logDirectory</td>
    <td>file:/tmp/spark-events</td>
    <td>
      为了文件系统的历史提供者，包含要加载的应用程序事件日志的目录URL。
      这可以是local <code>file://</code>  路径，
      HDFS <code>hdfs://namenode/shared/spark-logs</code> 
      或者是 Hadoop API支持的替代文件系统。
    </td>
  </tr>
  <tr>
    <td>spark.history.fs.update.interval</td>
    <td>10s</td>
    <td>
      文件系统历史的提供者在日志目录中检查新的或更新的日志期间。
      更短的时间间隔可以更快地检测新的应用程序，而不必更多服务器负载重新读取更新的应用程序。
      一旦更新完成，完成和未完成的应用程序的列表将反映更改。
    </td>
  </tr>
  <tr>
    <td>spark.history.retainedApplications</td>
    <td>50</td>
    <td>
      在缓存中保留UI数据的应用程序数量。 
      如果超出此上限，则最早的应用程序将从缓存中删除。 
      如果应用程序不在缓存中，如果从UI 界面访问它将不得不从磁盘加载。
    </td>
  </tr>
  <tr>
    <td>spark.history.ui.maxApplications</td>
    <td>Int.MaxValue</td>
    <td>
      在历史记录摘要页面上显示的应用程序数量。
      应用程序UI仍然可以通过直接访问其URL，即使它们不显示在历史记录摘要页面上。
    </td>
  </tr>
  <tr>
    <td>spark.history.ui.port</td>
    <td>18080</td>
    <td>
      history server 的Web界面绑定的端口。
    </td>
  </tr>
  <tr>
    <td>spark.history.kerberos.enabled</td>
    <td>false</td>
    <td>
      表明 history server 是否应该使用kerberos进行登录。
      如果 history server 正在访问安全的Hadoop集群上的HDFS文件，则需要这样做。
      如果这是真的，它使用配置 <code>spark.history.kerberos.principal</code> 和 <code>spark.history.kerberos.keytab</code>
    </td>
  </tr>
  <tr>
    <td>spark.history.kerberos.principal</td>
    <td>(none)</td>
    <td>
      history server 的Kerberos主要名称。
    </td>
  </tr>
  <tr>
    <td>spark.history.kerberos.keytab</td>
    <td>(none)</td>
    <td>
      history server 的kerberos keytab文件的位置。
    </td>
  </tr>
  <tr>
    <td>spark.history.ui.acls.enable</td>
    <td>false</td>
    <td>
      指定是否应检查acls授权查看应用程序的用户。
      如果启用，则进行访问控制检查，无论单个应用程序在运行时为 <code>spark.ui.acls.enable</code> 设置了什么。
      应用程序所有者将始终有权查看自己的应用程序和通过 <code>spark.ui.view.acls</code> 指定的任何用户和通过<code>spark.ui.view.acls.groups</code>，当应用程序运行时也将有权查看该应用程序。
      如果禁用，则不进行访问控制检查。
    </td>
  </tr>
  <tr>
    <td>spark.history.ui.admin.acls</td>
    <td>empty</td>
    <td>
      通过逗号来分隔具有对history server中所有Spark应用程序的查看访问权限的用户/管理员列表。
      默认情况下只允许在运行时查看应用程序的用户可以访问相关的应用程序历史记录，配置的用户/管理员也可以具有访问权限。
      在列表中添加 "*" 表示任何用户都可以拥有管理员的权限。
    </td>
  </tr>
  <tr>
    <td>spark.history.ui.admin.acls.groups</td>
    <td>empty</td>
    <td>
      通过逗号来分隔具有对history server中所有Spark应用程序的查看访问权限的组的列表。
      默认情况下只允许在运行时查看应用程序的组可以访问相关的应用程序历史记录，配置的组也可以具有访问权限。
      在列表中放置 "*" 表示任何组都可以拥有管理员权限。
    </td>
  </tr>
  <tr>
    <td>spark.history.fs.cleaner.enabled</td>
    <td>false</td>
    <td>
      指定 History Server是否应该定期从存储中清除事件日志。
    </td>
  </tr>
  <tr>
    <td>spark.history.fs.cleaner.interval</td>
    <td>1d</td>
    <td>
      文件系统 job history清洁程序多久检查要删除的文件。
      如果文件比 <code>spark.history.fs.cleaner.maxAge</code> 更旧，那么它们将被删除。
    </td>
  </tr>
  <tr>
    <td>spark.history.fs.cleaner.maxAge</td>
    <td>7d</td>
    <td>
      较早的Job history文件将在文件系统历史清除程序运行时被删除。
    </td>
  </tr>
  <tr>
    <td>spark.history.fs.numReplayThreads</td>
    <td>25% of available cores</td>
    <td>
      history server 用于处理事件日志的线程数。
    </td>
  </tr>
</table>

请注意UI中所有的任务，表格可以通过点击它们的标题来排序，便于识别慢速任务，数据偏移等。

注意

1. history server 显示完成的和未完成的Spark作业。
如果应用程序在失败后进行多次尝试，将显示失败的尝试，以及任何持续未完成的尝试或最终成功的尝试。

2. 未完成的程序只会间歇性地更新。
更新的时间间隔由更改文件的检查间隔 (`spark.history.fs.update.interval`) 定义。
在较大的集群上，更新间隔可能设置为较大的值。
查看正在运行的应用程序的方式实际上是查看自己的Web UI。

3. 没有注册完成就退出的应用程序将被列出为未完成的，即使它们不再运行。如果应用程序崩溃，可能会发生这种情况。

4. 一个用于表示完成Spark作业的一种方法是明确地停止Spark Context (`sc.stop()`)，或者在Python中使用  `with SparkContext() as sc:` 构造处理Spark上下文设置并拆除。


## REST API

除了在UI中查看指标之外，还可以使用JSON。
这为开发人员提供了一种简单的方法来为Spark创建新的可视化和监控工具。
JSON可用于运行的应用程序和 history server。The endpoints are mounted at `/api/v1`。
例如，对于 history server，它们通常可以在 `http://<server-url>:18080/api/v1` 访问，对于正在运行的应用程序，在 `http://localhost:4040/api/v1`。

在API中，一个应用程序被其应用程序ID `[app-id]`引用。
当运行在YARN上时，每个应用程序可能会有多次尝试，但是仅针对群集模式下的应用程序进行尝试，而不是客户端模式下的应用程序。
YARN群集模式中的应用程序可以通过它们的 `[attempt-id]`来识别。
在下面列出的API中，当以YARN集群模式运行时，`[app-id]`实际上是 `[base-app-id]/[attempt-id]`，其中 `[base-app-id]`YARN应用程序ID。

<table class="table">
  <tr><th>Endpoint</th><th>含义</th></tr>
  <tr>
    <td><code>/applications</code></td>
    <td>所有应用程序的列表。
    <br>
    <code>?status=[completed|running]</code> 列出所选状态下的应用程序。
    <br>
    <code>?minDate=[date]</code> 列出最早的开始日期/时间。
    <br>
    <code>?maxDate=[date]</code> 列出最新开始日期/时间。
    <br>
    <code>?minEndDate=[date]</code> 列出最早的结束日期/时间。
    <br>
    <code>?maxEndDate=[date]</code> 列出最新结束日期/时间。
    <br>
    <code>?limit=[limit]</code> 限制列出的应用程序数量。
    <br>示例:
    <br><code>?minDate=2015-02-10</code>
    <br><code>?minDate=2015-02-03T16:42:40.000GMT</code>
    <br><code>?maxDate=2015-02-11T20:41:30.000GMT</code>
    <br><code>?minEndDate=2015-02-12</code>
    <br><code>?minEndDate=2015-02-12T09:15:10.000GMT</code>
    <br><code>?maxEndDate=2015-02-14T16:30:45.000GMT</code>
    <br><code>?limit=10</code></td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/jobs</code></td>
    <td>
      给定应用程序的所有job的列表。
      <br><code>?status=[running|succeeded|failed|unknown]</code> 列出在特定状态下的job。
    </td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/jobs/[job-id]</code></td>
    <td>给定job的详细信息。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/stages</code></td>
    <td>
      给定应用程序的所有阶段的列表。
      <br><code>?status=[active|complete|pending|failed]</code> 仅列出状态的阶段。
    </td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/stages/[stage-id]</code></td>
    <td>
      给定阶段的所有尝试的列表。
    </td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]</code></td>
    <td>给定阶段的尝试详细信息。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskSummary</code></td>
    <td>
      给定阶段尝试中所有任务的汇总指标。
      <br><code>?quantiles</code> 用给定的分位数总结指标。
      <br>Example: <code>?quantiles=0.01,0.5,0.99</code>
    </td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskList</code></td>
    <td>
       给定阶段尝试的所有task的列表。
      <br><code>?offset=[offset]&amp;length=[len]</code> 列出给定范围内的task。
      <br><code>?sortBy=[runtime|-runtime]</code> task排序.
      <br>Example: <code>?offset=10&amp;length=50&amp;sortBy=runtime</code>
    </td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/executors</code></td>
    <td>给定应用程序的所有活动executor的列表。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/allexecutors</code></td>
    <td>给定应用程序的所有（活动和死亡）executor的列表。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/storage/rdd</code></td>
    <td>给定应用程序的存储RDD列表。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/storage/rdd/[rdd-id]</code></td>
    <td>给定RDD的存储状态的详细信息。</td>
  </tr>
  <tr>
    <td><code>/applications/[base-app-id]/logs</code></td>
    <td>将给定应用程序的所有尝试的事件日志下载为一个zip文件。</td>
  </tr>
  <tr>
    <td><code>/applications/[base-app-id]/[attempt-id]/logs</code></td>
    <td>将特定应用程序尝试的事件日志下载为一个zip文件。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/streaming/statistics</code></td>
    <td>streaming context的统计信息</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/streaming/receivers</code></td>
    <td>所有streaming receivers的列表。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/streaming/receivers/[stream-id]</code></td>
    <td>给定receiver的详细信息。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/streaming/batches</code></td>
    <td>所有被保留batch的列表。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/streaming/batches/[batch-id]</code></td>
    <td>给定batch的详细信息。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/streaming/batches/[batch-id]/operations</code></td>
    <td>给定batch的所有输出操作的列表。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/streaming/batches/[batch-id]/operations/[outputOp-id]</code></td>
    <td>给定操作和给定batch的详细信息。</td>
  </tr>
  <tr>
    <td><code>/applications/[app-id]/environment</code></td>
    <td>给定应用程序环境的详细信息。</td>
  </tr>       
</table>

可检索的 job 和 stage 的数量被standalone Spark UI 的相同保留机制所约束。
`"spark.ui.retainedJobs"` 定义触发job垃圾收集的阈值，以及 `spark.ui.retainedStages` 限定stage。
请注意，垃圾回收在play时进行：可以通过增加这些值并重新启动history server来检索更多条目。


### API 版本控制策略

这些 endpoint已被强力版本化，以便更容易开发应用程序。
特别是Spark保证：

* endpoint 永远不会从一个版本中删除
* 任何给定 endpoint都不会删除个别字段
* 可以添加新的 endpoint
* 可以将新字段添加到现有 endpoint
* 将来可能会在单独的 endpoint添加新版本的api（例如: `api/v2`）。 新版本 *不* 需要向后兼容。
* Api版本可能会被删除，但只有在至少一个与新的api版本共存的次要版本之后才可以删除。

请注意，即使在检查正在运行的应用程序的UI时，仍然需要 `applications/[app-id]`部分，尽管只有一个应用程序可用。
例如：要查看正在运行的应用程序的作业列表，您可以访问 `http://localhost:4040/api/v1/applications/[app-id]/jobs`。
这是为了在两种模式下保持路径一致。

# Metrics

Spark具有基于[Dropwizard Metrics Library](http://metrics.dropwizard.io/)的可配置metrics系统。
这允许用户将Spark metrics报告给各种接收器，包括HTTP，JMX和CSV文件。
metrics系统是通过配置文件进行配置的，Spark配置文件是Spark预计出现在 `$SPARK_HOME/conf/metrics.properties`上。 可以通过`spark.metrics.conf` [配置属性](configuration.html#spark-properties)指定自定义文件位置。
默认情况下，用于 driver或 executor metrics标准的根命名空间是 `spark.app.id`的值。
然而，通常用户希望能够跟踪 driver和executors的应用程序的metrics，这与应用程序ID（即：`spark.app.id`）很难相关，因为每次调用应用程序都会发生变化。
对于这种用例，可以为使用 `spark.metrics.namespace`配置属性的metrics报告指定自定义命名空间。
例如，如果用户希望将度量命名空间设置为应用程序的名称，则可以将`spark.metrics.namespace`属性设置为像 `${spark.app.name}`这样的值。 
然后，该值会被Spark适当扩展，并用作度量系统的根命名空间。
非 driver和 executor的metrics标准永远不会以 `spark.app.id`为前缀，`spark.metrics.namespace`属性也不会对这些metrics有任何这样的影响。

Spark的metrics被分解为与Spark组件相对应的不同_instances_。
在每个实例中，您可以配置一组报告汇总指标。
目前支持以下实例：

* `master`: Spark standalone的 master进程。
* `applications`: 主机内的一个组件，报告各种应用程序。
* `worker`: Spark standalone的 worker进程。
* `executor`: A Spark executor.
* `driver`: Spark driver进程（创建SparkContext的过程）。
* `shuffleService`: The Spark shuffle service.

每个实例可以报告为 0 或更多 _sinks_。 Sinks包含在 `org.apache.spark.metrics.sink`包中：

* `ConsoleSink`: 将metrics信息记录到控制台。
* `CSVSink`: 定期将metrics数据导出到CSV文件。
* `JmxSink`: 注册在JMX控制台中查看的metrics。
* `MetricsServlet`: 在现有的Spark UI中添加一个servlet，以将数据作为JSON数据提供。
* `GraphiteSink`: 将metrics发送到Graphite节点。
* `Slf4jSink`: 将metrics标准作为日志条目发送到slf4j。

Spark还支持由于许可限制而不包含在默认构建中的Ganglia接收器：

* `GangliaSink`: 向Ganglia节点或 multicast组发送metrics。

要安装 `GangliaSink` ，您需要执行Spark的自定义构建。
_**请注意，通过嵌入此库，您将包括 [LGPL](http://www.gnu.org/copyleft/lesser.html)-licensed Spark包中的代码**_。
对于sbt用户，在构建之前设置 `SPARK_GANGLIA_LGPL`环境变量。 对于Maven用户，启用 `-Pspark-ganglia-lgpl`配置文件。 
除了修改集群的Spark构建用户，应用程序还需要链接到 `spark-ganglia-lgpl`工件。

metrics配置文件的语法在示例配置文件 `$SPARK_HOME/conf/metrics.properties.template`中定义。

# 高级工具

可以使用几种外部工具来帮助描述Spark job的性能：

* 集群范围的监控工具，例如 [Ganglia](http://ganglia.sourceforge.net/)可以提供对整体集群利用率和资源瓶颈的洞察。例如，Ganglia仪表板可以快速显示特定工作负载是否为磁盘绑定，网络绑定或CPU绑定。
* 操作系统分析工具，如 [dstat](http://dag.wieers.com/home-made/dstat/)，[iostat](http://linux.die.net/man/1/iostat) 和 [iotop](http://linux.die.net/man/1/iotop) 可以在单个节点上提供细粒度的分析。
* JVM实用程序，如 `jstack` 提供堆栈跟踪，`jmap`用于创建堆转储，`jstat`用于报告时间序列统计数据和`jconsole`用于可视化地浏览各种JVM属性对于那些合适的JVM内部使用是有用的。
