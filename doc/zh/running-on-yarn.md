---
layout: global
title: Running Spark on YARN
---

支持在 [YARN (Hadoop
NextGen)](http://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/YARN.html)
上运行是在 Spark 0.6.0 版本中加入到 Spark 中的，并且在后续的版本中得到改进的。

# 启动 Spark on YARN

确保 `HADOOP_CONF_DIR` 或者 `YARN_CONF_DIR` 指向包含 Hadoop 集群的（客户端）配置文件的目录。这些配置被用于写入 HDFS 并连接到 YARN ResourceManager 。此目录中包含的配置将被分发到 YARN 集群，以便 application（应用程序）使用的所有的所有 containers（容器）都使用相同的配置。如果配置引用了 Java 系统属性或者未由 YARN 管理的环境变量，则还应在 Spark 应用程序的配置（driver（驱动程序），executors（执行器），和在客户端模式下运行时的 AM ）。

有两种部署模式可以用于在 YARN 上启动 Spark 应用程序。在 `cluster` 集群模式下， Spark driver 运行在集群上由 YARN 管理的application master 进程内，并且客户端可以在初始化应用程序后离开。在 `client` 客户端模式下，driver 在客户端进程中运行，并且 application master 仅用于从 YARN 请求资源。


与 [Spark standalone](spark-standalone.html) 和 [Mesos](running-on-mesos.html) 不同的是，在这两种模式中，master 的地址在 `--master` 参数中指定，在 YARN 模式下， ResourceManager 的地址从 Hadoop 配置中选取。因此， `--master` 参数是 `yarn` 。

在 `cluster` 集群模式下启动 Spark 应用程序:

    $ ./bin/spark-submit --class path.to.your.Class --master yarn --deploy-mode cluster [options] <app jar> [app options]

例如:

    $ ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
        --master yarn \
        --deploy-mode cluster \
        --driver-memory 4g \
        --executor-memory 2g \
        --executor-cores 1 \
        --queue thequeue \
        lib/spark-examples*.jar \
        10

上面启动一个 YARN 客户端程序，启动默认的 Application Master。然后 SparkPi 将作为 Application Master 的子进程运行。客户端将定期轮询 Application Master 以获取状态的更新并在控制台中显示它们。一旦您的应用程序完成运行后，客户端将退出。请参阅下面的 "调试应用程序" 部分，了解如何查看 driver 和 executor 的日志。

要在 `client` 客户端模式下启动 Spark 应用程序，请执行相同的操作，但是将 `cluster` 替换 `client` 。下面展示了如何在 `client` 客户端模式下运行 `spark-shell`: 


    $ ./bin/spark-shell --master yarn --deploy-mode client

## 添加其他的 JARs

在 `cluster` 集群模式下，driver 在与客户端不同的机器上运行，因此 `SparkContext.addJar` 将不会立即使用客户端本地的文件运行。要使客户端上的文件可用于 `SparkContext.addJar` ，请在启动命令中使用 `--jars` 选项来包含这些文件。


    $ ./bin/spark-submit --class my.main.Class \
        --master yarn \
        --deploy-mode cluster \
        --jars my-other-jar.jar,my-other-other-jar.jar \
        my-main-jar.jar \
        app_arg1 app_arg2


# 准备

在 YARN 上运行 Spark 需要使用 YARN 支持构建的二进制分布式的 Spark （a binary distribution of Spark）。二进制文件（binary distributions）可以从项目网站的 [下载页面](http://spark.apache.org/downloads.html) 下载。要自己构建 Spark ，请参考 [构建 Spark](building-spark.html). 。


要使 Spark 运行时 jars 可以从 YARN 端访问，您可以指定 `spark.yarn.archive` 或者 `spark.yarn.jars` 。更多详细的信息，请参阅 [Spark 属性](running-on-yarn.html#spark-properties) 。如果既没有指定 `spark.yarn.archive` 也没有指定 `spark.yarn.jars` ，Spark 将在 `$SPARK_HOME/jars` 目录下创建一个包含所有 jar 的 zip 文件，并将其上传到 distributed cache（分布式缓存）中。


# 配置

对于 Spark on YARN 和其他的部署模式，大多数的配置是相同的。有关这些的更多信息，请参阅 [配置页面](configuration.html) 。这些是特定于 YARN 上的 Spark 的配置。


# 调试应用

在 YARN 术语中，executors（执行器）和 application masters 在 "containers（容器）" 中运行时。 YARN 有两种模式用于在应用程序完成后处理容器日志（container logs）。如果启用日志聚合（aggregation）（使用 `yarn.log-aggregation-enable` 配置），容器日志（container logs）将复制到 HDFS 并在本地计算机上删除。可以使用 `yarn logs` 命令从集群中的任何位置查看这些日志。

    yarn logs -applicationId <app ID>

将打印来自给定的应用程序的所有容器（containers）的所有的日志文件的内容。你还可以使用 HDFS shell 或者 API 直接在 HDFS 中查看容器日志文件（container log files）。可以通过查看您的 YARN 配置（`yarn.nodemanager.remote-app-log-dir` 和 `yarn.nodemanager.remote-app-log-dir-suffix`）找到它们所在的目录。日志还可以在 Spark Web UI 的 “执行程序（Executors）”选项卡下找到。您需要同时运行 Spark 历史记录服务器（Spark history server） 和 MapReduce 历史记录服务器（MapReduce history server），并在 `yarn-site.xm`l 文件中正确配置 `yarn.log.server.url`。Spark 历史记录服务器 UI 上的日志将重定向您到 MapReduce 历史记录服务器以显示聚合日志（aggregated logs）。

当未启用日志聚合时，日志将在每台计算机上的本地保留在 `YARN_APP_LOGS_DIR 目录下`，通常配置为 `/tmp/logs` 或者 `$HADOOP_HOME/logs/userlogs` ，具体取决于 Hadoop 版本和安装。查看容器（container）的日志需要转到包含它们的主机并在此目录中查看它们。子目录根据应用程序 ID （application ID）和 容器 ID （container ID）组织日志文件。日志还可以在 Spark Web UI 的 “执行程序（Executors）”选项卡下找到，并且不需要运行 MapReduce  history server。

要查看每个 container（容器）的启动环境，请将 `yarn.nodemanager.delete.debug-delay-sec` 增加到一个较大的值（例如 `36000`），然后通过 `yarn.nodemanager.local-dirs` 访问应用程序缓存，在容器启动的节点上。此目录包含启动脚本（launch script）， JARs ，和用于启动每个容器的所有的环境变量。这个过程对于调试 classpath 问题特别有用。（请注意，启用此功能需要集群设置的管理员权限并且还需要重新启动所有的 node managers，因此这不适用于托管集群）。

要为 application master 或者 executors 使用自定义的 log4j 配置，请选择以下选项:

- 使用 `spark-submit` 上传一个自定义的 `log4j.properties` ，通过将 spark-submit 添加到要与应用程序一起上传的文件的      --files 列表中。
- add `-Dlog4j.configuration=<location of configuration file>` to `spark.driver.extraJavaOptions`
  (for the driver) or `spark.executor.extraJavaOptions` (for executors). Note that if using a file,
  the `file:` protocol should be explicitly provided, and the file needs to exist locally on all
  the nodes.
- 添加 `-Dlog4j.configuration=<配置文件的位置>` 到 `spark.driver.extraJavaOptions`（对于驱动程序）或者 containers （对于执行者）。请注意，如果使用文件，文件: 协议（protocol ）应该被显式提供，并且该文件需要在所有节点的本地存在。
- 更新 `$SPARK_CONF_DIR/log4j.properties` 文件，并且它将与其他配置一起自动上传。请注意，如果指定了多个选项，其他 2 个选项的优先级高于此选项。

请注意，对于第一个选项，executors 和 application master 将共享相同的 log4j 配置，这当它们在同一个节点上运行的时候，可能会导致问题（例如，试图写入相同的日志文件）。

如果你需要引用正确的位置将日志文件放在 YARN 中，以便 YARN 可以正确显示和聚合它们，请在您的 `log4j.properties` 中使用 `spark.yarn.app.container.log.dir`。例如，`log4j.appender.file_appender.File=${spark.yarn.app.container.log.dir}/spark.log`。对于流应用程序（streaming applications），配置 `RollingFileAppender` 并将文件位置设置为 YARN 的日志目录将避免由于大型日志文件导致的磁盘溢出，并且可以使用 YARN 的日志实用程序（YARN’s log utility）访问日志。

To use a custom metrics.properties for the application master and executors, update the `$SPARK_CONF_DIR/metrics.properties` file. It will automatically be uploaded with other configurations, so you don't need to specify it manually with `--files`.

要为 application master 和 executors 使用一个自定义的 metrics.properties，请更新 `$SPARK_CONF_DIR/metrics.properties` 文件。它将自动与其他配置一起上传，因此您不需要使用 `--files` 手动指定它。

#### Spark 属性

<table class="table">
<tr><th>Property Name（属性名称）</th><th>Default（默认值）</th><th>Meaning（含义）</th></tr>
<tr>
  <td><code>spark.yarn.am.memory</code></td>
  <td><code>512m</code></td>
  <td>
    在客户端模式下用于 YARN Application Master 的内存量，与 JVM 内存字符串格式相同 (e.g. <code>512m</code>, <code>2g</code>).
    在 `cluster` 集群模式下, 使用 <code>spark.driver.memory</code> 代替.
    <p/>
    使用小写字母尾后缀, 例如. <code>k</code>, <code>m</code>, <code>g</code>, <code>t</code>, and <code>p</code>, 分别表示 kibi-, mebi-, gibi-, tebi- 和 pebibytes, respectively.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.am.cores</code></td>
  <td><code>1</code></td>
  <td>
    在客户端模式下用于 YARN Application Master 的核数。在集群模式下，请改用 <code>spark.driver.cores</code> 代替.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.am.waitTime</code></td>
  <td><code>100s</code></td>
  <td>
    在 <code>cluster 集群</code> 模式中, YARN Application Master 等待
    SparkContext 被初始化的时间. 在 <code>client 客户端</code> 模式中, YARN Application Master 等待 driver 连接它的时间.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.submit.file.replication</code></td>
  <td>默认的 HDFS 副本数 (通常是 <code>3</code>)</td>
  <td>
    用于应用程序上传到 HDFS 的文件的 HDFS 副本级别。这些包括诸如 Spark jar ，app jar， 和任何分布式缓存 files/archives
之类的东西。
  </td>
</tr>
<tr>
  <td><code>spark.yarn.stagingDir</code></td>
  <td>文件系统中当前用户的主目录</td>
  <td>
    提交应用程序时使用的临时目录.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.preserve.staging.files</code></td>
  <td><code>false</code></td>
  <td>
    设置为 <code>true</code> 以便在作业结束保留暂存文件（Spark jar， app jar，分布式缓存文件），而不是删除它们.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.scheduler.heartbeat.interval-ms</code></td>
  <td><code>3000</code></td>
  <td>
    Spark application master 心跳到 YARN ResourceManager 中的间隔（以毫秒为单位）。该值的上限为到期时间间隔的 YARN
配置值的一半，即 <code>yarn.am.liveness-monitor.expiry-interval-ms</code>.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.scheduler.initial-allocation.interval</code></td>
  <td><code>200ms</code></td>
  <td>
    当存在未决容器分配请求时，Spark application master 心跳到 YARN ResourceManager 的初始间隔。它不应大于
    <code>spark.yarn.scheduler.heartbeat.interval-ms</code>. 如果挂起的容器仍然存在，直到达到
    <code>spark.yarn.scheduler.heartbeat.interval-ms</code> 为止.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.max.executor.failures</code></td>
  <td>numExecutors * 2，最小值为 3/td>
  <td>
    在应用程序失败（failing the application）之前，执行器失败（executor failures）的最大数量.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.historyServer.address</code></td>
  <td>(none)</td>
  <td>
    默认为未设置，因为 history server 是可选服务。当 Spark 应用程序完成将应用程序从 ResourceManager UI 链接到 Spark history server UI 时，此地址将提供给 YARN ResourceManager。对于此属性，YARN 属性可用作变量，这些属性在运行时由 Spark 替换。例如，如果 Spark 历史记录服务器在与 YARN ResourceManager 相同的节点上运行，则可以将其设置为 <code>${hadoopconf-yarn.resourcemanager.hostname}:18080</code>.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.dist.archives</code></td>
  <td>(none)</td>
  <td>
    逗号分隔的要提取到每个执行器（executor）的工作目录（working directory）中的归档列表。
  </td>
</tr>
<tr>
  <td><code>spark.yarn.dist.files</code></td>
  <td>(none)</td>
  <td>
    C要放到每个执行器（executor）的工作目录（working directory）中的以逗号分隔的文件列表.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.dist.jars</code></td>
  <td>(none)</td>
  <td>
    要放到每个执行器（executor）的工作目录（working directory）中的以逗号分隔的 jar 文件列表.
  </td>
</tr>
<tr>
 <td><code>spark.executor.instances</code></td>
  <td><code>2</code></td>
  <td>
    静态分配的执行器（executor）数量。使用 <code>spark.dynamicAllocation.enabled</code>，初始的执行器（executor）数量至少会是这么大, .
  </td>
</tr>
<tr>
 <td><code>spark.yarn.executor.memoryOverhead</code></td>
  <td>executorMemory * 0.10, 最小值是 384 </td>
  <td>
    要为每个执行器（executor）分配的堆外（off-heap）内存量（以兆字节为单位）。这是内存，例如 VM 开销，内部字符串，其他本机开销等。这往往随着执行器（executor）大小（通常为 6-10%）增长.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.driver.memoryOverhead</code></td>
  <td>driverMemory * 0.10, 最小值是 384 </td>
  <td>
    在集群模式下为每个驱动程序（driver）分配的堆外（off-heap）内存量（以兆字节为单位）。这是内存，例如 VM 开销，内部字符串，其他本机开销等。这往往随着容器（container）大小（通常为 6- 10%）增长.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.am.memoryOverhead</code></td>
  <td>AM memory * 0.10, 最小值是 384 </td>
  <td>
    与 <code>spark.yarn.driver.memoryOverhead</code> 一样, 但是只适用于客户端模式下的 YARN Application Master.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.am.port</code></td>
  <td>(random)</td>
  <td>
    被 YARN Application Master 监听的端口。在 YARN 客户端模式下，这用于在网关（gateway）上运行的 Spark 驱动程序（driver）和在 YARN 上运行的 YARN Application Master 之间进行通信。在 YARN 集群模式下，这用于动态执行器（executor）功能，其中它处理从调度程序后端的 kill.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.queue</code></td>
  <td><code>default</code></td>
  <td>
    提交应用程序（application）的 YARN 队列名称.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.jars</code></td>
  <td>(none)</td>
  <td>
    包含要分发到 YARN 容器（container）的 Spark 代码的库（libraries）列表。默认情况下， Spark on YARN 将使用本地安装的 Spark jar，但是 Spark jar 也可以在 HDFS 上的一个任何位置都可读的位置。这允许 YARN 将其缓存在节点上，使得它不需要在每次运行应用程序时分发。例如，要指向 HDFS 上的 jars ，请将此配置设置为 <code>hdfs:///some/path</code>。允许使用 globs.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.archive</code></td>
  <td>(none)</td>
  <td>
    包含所需的 Spark jar 的归档（archive），以分发到 YARN 高速缓存。如果设置，此配置将替换 <code>spark.yarn.jars</code>，并且归档（archive）在所有的应用程序（application）的容器（container）中使用。归档（archive）应该在其根目录中包含 jar 文件。与以前的选项一样，归档也可以托管在 HDFS 上以加快文件分发速度.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.access.hadoopFileSystems</code></td>
  <td>(none)</td>
  <td>
    应用程序将要访问的安全 Hadoop 文件系统的逗号分隔列表。 例如, <code>spark.yarn.access.hadoopFileSystems=hdfs://nn1.com:8032,hdfs://nn2.com:8032,
    webhdfs://nn3.com:50070</code>. 应用程序必须能够访问列出的文件系统并且 Kerberos 必须被正确地配置为能够访问它们（在同一个 realm 域或在受信任的 realm 域）。Spark 为每个文件系统获取安全的 tokens 令牌 Spark 应用程序可以访问这些远程 Hadoop 文件系统。Spark <code>spark.yarn.access.namenodes</code> 已经过时了, 请使用这个来代替.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.appMasterEnv.[EnvironmentVariableName]</code></td>
  <td>(none)</td>
  <td>
     将由 <code>EnvironmentVariableName</code> 指定的环境变量添加到在 YARN 上启动的 Application Master 进程。用户可以指定其中的多个并设置多个环境变量。在 <code>cluster 集群</code> 模式下，这控制 Spark 驱动程序（driver）的环境，在 <code>client 客户端</code> 模式下，它只控制执行器（executor）启动器（launcher）的环境.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.containerLauncherMaxThreads</code></td>
  <td><code>25</code></td>
  <td>
    在 YARN Application Master 中用于启动执行器容器（executor containers）的最大线程（thread）数.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.am.extraJavaOptions</code></td>
  <td>(none)</td>
  <td>
  在客户端模式下传递到 YARN Application Master 的一组额外的 JVM 选项。在集群模式下，请改用 <code>spark.driver.extraJavaOptions</code>。请注意，使用此选项设置最大堆大小（-Xmx）设置是非法的。最大堆大小设置可以使用 <code>spark.yarn.am.memory</code>.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.am.extraLibraryPath</code></td>
  <td>(none)</td>
  <td>
    设置在客户端模式下启动 YARN Application Master 时要使用的特殊库路径（special library path）.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.maxAppAttempts</code></td>
  <td><code>yarn.resourcemanager.am.max-attempts</code> in YARN</td>
  <td>
  将要提交应用程序的最大的尝试次数。它应该不大于 YARN 配置中的全局最大尝试次数。
  </td>
</tr>
<tr>
  <td><code>spark.yarn.am.attemptFailuresValidityInterval</code></td>
  <td>(none)</td>
  <td>
  定义 AM 故障跟踪（failure tracking）的有效性间隔。如果 AM 已经运行至少所定义的间隔，则 AM 故障计数将被重置。如果未配置此功能，则不启用此功能.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.executor.failuresValidityInterval</code></td>
  <td>(none)</td>
  <td>
  定义执行器（executor）故障跟踪的有效性间隔。超过有效性间隔的执行器故障（executor failures）将被忽略.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.submit.waitAppCompletion</code></td>
  <td><code>true</code></td>
  <td>
  在 YARN cluster 集群模式下，控制客户端是否等待退出，直到应用程序完成。如果设置为 <code>true</code>，客户端将保持报告应用程序的状态。否则，客户端进程将在提交后退出。

  </td>
</tr>
<tr>
  <td><code>spark.yarn.am.nodeLabelExpression</code></td>
  <td>(none)</td>
  <td>
  一个将调度限制节点 AM 集合的 YARN 节点标签表达式。只有大于或等于 2.6 版本的 YARN 版本支持节点标签表达式，因此在对较早版本运行时，此属性将被忽略。
  </td>
</tr>
<tr>
  <td><code>spark.yarn.executor.nodeLabelExpression</code></td>
  <td>(none)</td>
  <td>
  一个将调度限制节点执行器（executor）集合的 YARN 节点标签表达式。只有大于或等于 2.6 版本的 YARN 版本支持节点标签表达式，因此在对较早版本运行时，此属性将被忽略。

  </td>
</tr>
<tr>
  <td><code>spark.yarn.tags</code></td>
  <td>(none)</td>
  <td>
  以逗号分隔的字符串列表，作为 YARN application 标记中显示的 YARN application 标记传递，可用于在查询 YARN apps 时进行过滤（filtering ）.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.keytab</code></td>
  <td>(none)</td>
  <td>
  包含上面指定的主体（principal）的 keytab 的文件的完整路径。此 keytab 将通过安全分布式缓存（Secure Distributed Cache）复制到运行 YARN Application Master 的节点，以定期更新 login tickets 和 delegation tokens.（也可以在 "local" master 下工作）。
  </td>
</tr>
<tr>
  <td><code>spark.yarn.principal</code></td>
  <td>(none)</td>
  <td>
  在安全的 HDFS 上运行时用于登录 KDC 的主体（Principal ）. (也可以在 "local" master 下工作)
  </td>
</tr>
<tr>
  <td><code>spark.yarn.config.gatewayPath</code></td>
  <td>(none)</td>
  <td>
  一个在网关主机（gateway host）（启动 Spark application 的 host）上有效的路径，但对于集群中其他节点中相同资源的路径可能不同。结合
  <code>spark.yarn.config.replacementPath</code>, 这个用于支持具有异构配置的集群, 以便 Spark 可以正确启动远程进程.
  <p/>
  替换路径（replacement path）通常将包含对由 YARN 导出的某些环境变量（以及，因此对于 Spark 容器可见）的引用.
  <p/>
  例如，如果网关节点（gateway node）在 <code>/disk1/hadoop</code> 上安装了 Hadoop 库，并且 Hadoop 安装的位置由 YARN 作为 <code>HADOOP_HOME</code> 环境变量导出，则将此值设置为 <code>/disk1/hadoop</code> ，将替换路径（replacement path）设置为 <code>$HADOOP_HOME</code> 将确保用于启动远程进程的路径正确引用本地 YARN 配置。
  </td>
</tr>
<tr>
  <td><code>spark.yarn.config.replacementPath</code></td>
  <td>(none)</td>
  <td>
  请看 <code>spark.yarn.config.gatewayPath</code>.
  </td>
</tr>
<tr>
  <td><code>spark.yarn.security.credentials.${service}.enabled</code></td>
  <td><code>true</code></td>
  <td>
  控制是否在启用安全性时获取服务的 credentials（凭据）。默认情况下，在这些服务时，将检索所有支持的服务的凭据配置，但如果与某些方式冲突，可以禁用该行为. 详情请见 [在安全的集群中运行](running-on-yarn.html#running-in-a-secure-cluster)
  </td>
</tr>
<tr>
  <td><code>spark.yarn.rolledLog.includePattern</code></td>
  <td>(none)</td>
  <td>
  Java Regex 过滤与定义的包含模式匹配的日志文件，这些日志文件将以滚动的方式进行聚合。这将与 YARN 的滚动日志聚合一起使用，在 yarn-site.xml 文件中配置 <code>yarn.nodemanager.log-aggregation.roll-monitoring-interval-seconds</code> 以在 YARN 方面启用此功能.
  此功能只能与 Hadoop 2.6.4+ 一起使用。 Spark log4j appender 需要更改才能使用 FileAppender 或其他 appender 可以处理正在运行的文件被删除。基于在 log4j 配置中配置的文件名（如 spark.log）上，用户应该设置正则表达式（spark *）包含需要聚合的所有日志文件。
  </td>
</tr>
<tr>
  <td><code>spark.yarn.rolledLog.excludePattern</code></td>
  <td>(none)</td>
  <td>
  Java Regex 过滤与定义的排除模式匹配的日志文件，并且这些日志文件将不会以滚动的方式进行聚合。 如果日志文件名称匹配include 和 exclude 模式，最终将排除此文件。
  </td>
</tr>
</table>

# 重要提示

- core request 在调度决策中是否得到执行取决于使用的调度程序及其配置方式.
- 在 `cluster 集群` 模式中, Spark executors 和 Spark dirver 使用的本地目录是为 YARN（Hadoop YARN 配置 `yarn.nodemanager.local-dirs`）配置的本地目录. 如果用户指定 `spark.local.dir`, 它将被忽略. 在 `client 客户端` 模式下, Spark executors 将使用为 YARN 配置的本地目录, Spark dirver 将使用 `spark.local.dir` 中定义的目录.
这是因为 Spark drivers 不在 YARN cluster 的 `client ` 模式中运行，仅仅 Spark 的 executors 会这样做。
- `--files` 和 `--archives` 支持用 # 指定文件名，与 Hadoop 相似. 例如您可以指定: `--files localtest.txt#appSees.txt` 然后就会上传本地名为 `localtest.txt` 的文件到 HDFS 中去，但是会通过名称 `appSees.txt` 来链接, 当你的应用程序在 YARN 上运行时，你应该使用名称 `appSees.txt` 来引用它.
- The `--jars` 选项允许你在 `cluster 集群` 模式下使用本地文件时运行 `SparkContext.addJar` 函数. 如果你使用 HDFS，HTTP，HTTPS 或 FTP 文件，则不需要使用它.

# 在安全集群中运行

如 [security](security.html) 所讲的，Kerberos 被应用在安全的 Hadoop 集群中去验证与服务和客户端相关联的 principals。 这允许客户端请求这些已验证的服务; 向授权的 principals 授予请求服务的权利。

Hadoop 服务发出 *hadoop tokens*  去授权访问服务和数据。 客户端必须首先获取它们将要访问的服务的 tokens，当启动应用程序时，将它和应用程序一起发送到 YAYN 集群中。

如果 Spark 应用程序与其它任何的 Hadoop 文件系统（例如 hdfs，webhdfs，等等），HDFS，HBase 和 Hive 进行交互，它必须使用启动应用程序的用户的 Kerberos 凭据获取相关 tokens，也就是说身份将成为已启动的 Spark 应用程序的 principal。

这通常在启动时完成 : 在安全集群中，Spark 将自动为集群的 HDFS 文件系统获取 tokens，也可能为 HBase 和 Hive 获取。

如果 HBase 在 classpath 中，HBase token 是可以获取的，HBase 配置声明应用程序是安全的（即 `hbase-site.xml` 将 `hbase.security.authentication` 设置为 `kerberos`），并且 `spark.yarn.security.tokens.hbase.enabled` 未设置为 `false`，HBase tokens 将被获得。

类似地，如果 Hive 在 classpath 中，其配置包括元数据存储的 URI（`hive.metastore.uris`），并且 `spark.yarn.security.tokens.hive.enabled` 未设置为 `false`，则将获得 Hive token（令牌）。

如果应用程序需要与其他安全 Hadoop 文件系统交互，则在启动时必须显式请求访问这些集群所需的 tokens。 这是通过将它们列在 1spark.yarn.access.namenodes1 属性中来实现的。

```
spark.yarn.access.hadoopFileSystems hdfs://ireland.example.org:8020/,webhdfs://frankfurt.example.org:50070/
```

Spark 支持通过 Java Services 机制（请看 `java.util.ServiceLoader`）与其它的具有安全性的服务来进行集成。为了实现该目标，通过在 jar 的 `META-INF/services` 目录中列出相应 `org.apache.spark.deploy.yarn.security.ServiceCredentialProvider` 的实现的名字就可应用到 Spark。这些插件可以通过设置 `spark.yarn.security.tokens.{service}.enabled` 为 `false` 来禁用，这里的 `{service}` 是 credential provider（凭证提供者）的名字。


## 配置外部的 Shuffle Service

要在 YARN cluster 中的每个 `NodeManager` 中启动 Spark Shuffle，按照以下说明:

1. 用 [YARN profile](building-spark.html) 来构建 Spark. 如果你使用了预包装的发布可以跳过该步骤.
1. 定位 `spark-<version>-yarn-shuffle.jar`. 如果是你自己构建的 Spark，它应该在
`$SPARK_HOME/common/network-yarn/target/scala-<version>` 下, 如果你使用的是一个发布的版本，那么它应该在 `yarn` 下.
1. 添加这个 jar 到你集群中所有的 `NodeManager` 的 classpath 下去。
1. 在每个 node（节点）的 `yarn-site.xml` 文件中, 添加 `spark_shuffle` 到 `yarn.nodemanager.aux-services`,
然后设置 `yarn.nodemanager.aux-services.spark_shuffle.class` 为
`org.apache.spark.network.yarn.YarnShuffleService`.
1. 通过在 `etc/hadoop/yarn-env.sh` 文件中设置 `YARN_HEAPSIZE` (默认值 1000) 增加 `NodeManager's` 堆大小以避免在 shuffle 时的 garbage collection issues（垃圾回收问题）。
1. 重启集群中所有的 `NodeManager`.

当 shuffle service 服务在 YARN 上运行时，可以使用以下额外的配置选项:

<table class="table">
<tr><th>Property Name（属性名称）</th><th>Default（默认值）</th><th>Meaning（含义）</th></tr>
<tr>
  <td><code>spark.yarn.shuffle.stopOnFailure</code></td>
  <td><code>false</code></td>
  <td>
    是否在 Spark Shuffle Service 初始化出现故障时停止 NodeManager. This prevents application failures caused by running containers on
    NodeManagers where the Spark Shuffle Service is not running. 
  </td>
</tr>
</table>

## 用 Apache Oozie 来运行应用程序

Apache Oozie 可以将启动 Spark 应用程序作为工作流的一部分。在安全集群中，启动的应用程序将需要相关的 tokens 来访问集群的服务。如果 Spark 使用 keytab 启动，这是自动的。但是，如果 Spark 在没有 keytab 的情况下启动，则设置安全性的责任必须移交给 Oozie。

有关配置 Oozie 以获取安全集群和获取作业凭据的详细信息，请参阅 [Oozie web site](http://oozie.apache.org/) 上特定版本文档的 “Authentication” 部分。

对于 Spark 应用程序，必须设置 Oozie 工作流以使 Oozie 请求应用程序需要的所有 tokens，包括:

- The YARN resource manager.
- The local Hadoop filesystem.
- Any remote Hadoop filesystems used as a source or destination of I/O.
- Hive —if used.
- HBase —if used.
- The YARN timeline server, if the application interacts with this.

为了避免 Spark 尝试 - 然后失败 - 要获取 Hive，HBase 和远程的 HDFS 令牌，必须将 Spark 配置收集这些服务 tokens 的选项设置为禁用。

Spark 配置必须包含以下行:

```
spark.yarn.security.credentials.hive.enabled   false
spark.yarn.security.credentials.hbase.enabled  false
```

必须取消设置配置选项`spark.yarn.access.hadoopFileSystems`.

## Kerberos 故障排查

调试 Hadoop/Kerberos 问题可能是 “difficult 困难的”。 一个有用的技术是通过设置 `HADOOP_JAAS_DEBUG` 环境变量在 Hadoop 中启用对 Kerberos 操作的额外记录。

```bash
export HADOOP_JAAS_DEBUG=true
```

JDK 类可以配置为通过系统属性 `sun.security.krb5.debug` 和 `sun.security.spnego.debug=true` 启用对 Kerberos 和 SPNEGO/REST 认证的额外日志记录。

```
-Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true
```

所有这些选项都可以在 Application Master 中启用:

```
spark.yarn.appMasterEnv.HADOOP_JAAS_DEBUG true
spark.yarn.am.extraJavaOptions -Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true
```

最后，如果 `org.apache.spark.deploy.yarn.Client` 的日志级别设置为 `DEBUG`，日志将包括获取的所有 tokens 的列表，以及它们的到期详细信息。

## 使用 Spark History Server 来替换 Spark Web UI

当应用程序 UI 被禁用时的应用程序，可以使用 Spark History Server 应用程序页面作为运行程序用于跟踪的 URL。这在 secure clusters（安全的集群）中是适合的，或者减少 Spark driver 的内存使用量。 要通过 Spark History Server 设置跟踪，请执行以下操作:

- 在 application（应用）方面, 在 Spark 的配置中设置 <code>spark.yarn.historyServer.allowTracking=true</code>. 在 application's UI 是禁用的情况下，这将告诉 Spark 去使用 history server's URL 作为 racking URL。
- 在 Spark History Server 方面, 添加 <code>org.apache.spark.deploy.yarn.YarnProxyRedirectFilter</code>
  到 <code>spark.ui.filters</code> 配置的中的 filters 列表中去.

请注意，history server 信息可能不是应用程序状态的最新信息。
