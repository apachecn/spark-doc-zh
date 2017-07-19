---
layout: global
title: 在 Mesos 上运行 Spark
---
* This will become a table of contents (this text will be scraped).
{:toc}

Spark 可以运行在 [Apache Mesos](http://mesos.apache.org/) 管理的硬件集群上。

使用 Mesos 部署 Spark 的优点包括：

- Spark 与其他的 [frameworks（框架）](https://mesos.apache.org/documentation/latest/mesos-frameworks/) 之间的 dynamic partitioning （动态分区）
- 在 Spark 的多个实例之间的 scalable partitioning （可扩展分区）

# 运行原理

在一个 standalone 集群部署中，下图中的集群的 manager 是一个 Spark master 实例。当使用 Mesos 时，Mesos master 会将 Spark master 替换掉，成为集群 manager 。

<p style="text-align: center;">
  <img src="img/cluster-overview.png" title="Spark 集群组件" alt="Spark 集群组件" />
</p>

现在当 driver 创建一个作业并开始执行调度任务时， Mesos 会决定什么机器处理什么任务。因为 Mesos 调度这些短期任务时会将其他的框架考虑在内，多个框架可以共存在同一个集群上，而不需要求助于一个 static partitioning of resources （资源的静态分区）。

要开始，请按照以下步骤安装 Mesos 并通过 Mesos 部署 Spark 作业。


# 安装 Mesos

Spark {{site.SPARK_VERSION}} 专门为 Mesos {{site.MESOS_VERSION}} 或更新的版本并且不需要 Mesos 的任何特殊补丁。

如果您已经有一个 Mesos 集群正在运行着，您可以跳过这个 Mesos 安装的步骤。

否则，安装 Mesos for Spark 与安装 Mesos for 其他框架是没有区别的。您可以通过源码或者 prebuilt packages （预构建软件安装包）来安装 Mesos 。

## 从源码安装

通过源码安装 Apache Mesos ，请按照以下步骤：

1. 从镜像网站 [mirror](http://www.apache.org/dyn/closer.lua/mesos/{{site.MESOS_VERSION}}/) 下载一个 Mesos 发行版。
2. 按照 Mesos 的 [快速开始](http://mesos.apache.org/gettingstarted) 页面来编译和安装 Mesos 。

**注意:** 如果您希望运行 Mesos 并且又不希望将其安装在您的系统的默认路径（例如，如果您没有安装它的管理权限），将 `--prefix` 选项传入 `configure` 来告知它安装在什么地方。例如，将 `--prefix=/home/me/mesos` 传入。默认情况下，前缀是 `/usr/local` 。

## 第三方软件包

Apache Mesos 只发布了源码的发行版本，而不是 binary packages （二进制包）。但是其他的第三方项目发布了 binary releases （二进制发行版本），可能对设置 Mesos 有帮助。

其中之一是 Mesosphere 。使用 Mesosphere 提供的 binary releases （二进制发行版本）安装 Mesos ：

1. 从 [下载页面](http://mesosphere.io/downloads/) 下载 Mesos 安装包
2. 按照他们的说明进行安装和配置

Mesosphere 安装文档建议安装 ZooKeeper 来处理 Mesos master 故障切换，但是 Mesos 可以在没有 ZooKeeper 的情况下使用 single master 。

## 验证

要验证 Mesos 集群是否已经准备好用于 Spark ，请导航到 Mesos master 的 webui 界面，端口是： `:5050` 来确认所有预期的机器都在 slaves 选项卡中。


# 连接 Spark 到 Mesos

要使用 Spark 中的 Mesos ，您需要一个 Spark 的二进制包放到 Mesos 可以访问的地方，然后一个 Spark driver 程序配置来连接 Mesos 。

或者，您也可以将 Spark 安装在所有 Mesos slaves 中的相同位置，并且配置 `spark.mesos.executor.home` （默认是 SPARK_HOME）来指向该位置。

## 上传 Spark 包

当 Mesos 第一次在 Mesos slave 上运行任务的时候，这个 slave 必须有一个 Spark binary
package （Spark 二进制包）用于执行 Spark Mesos executor backend （执行器后端）。
Spark 软件包可以在任何 Hadoop 可访问的 URI 上托管，包括 HTTP 通过 `http://`  ，[Amazon Simple Storage Service](http://aws.amazon.com/s3) 通过 `s3n://` ，或者 HDFS 通过 `hdfs://` 。

要使用预编译的包：

1. 从 Spark 的  [下载页面](https://spark.apache.org/downloads.html) 下载一个 Spark binary package （Spark 二进制包）
2. 上传到 hdfs/http/s3

要托管在 HDFS 上，使用 Hadoop fs put 命令：`hadoop fs -put spark-{{site.SPARK_VERSION}}.tar.gz
/path/to/spark-{{site.SPARK_VERSION}}.tar.gz`


或者如果您正在使用着一个自定义编译的 Spark 版本，您将需要使用 在 Spark 源码中的 tarball/checkout 的 `dev/make-distribution.sh` 脚本创建一个包。

1. 按照说明 [这里](index.html) 来下载并构建 Spark 。
2. 使用 `./dev/make-distribution.sh --tgz` 创建一个 binary package （二进制包）
3. 将归档文件上传到 http/s3/hdfs


## 使用 Mesos Master URL

对于一个 single-master Mesos 集群，Mesos 的 Master URLs 是以 `mesos://host:5050` 的形式表示的，或者对于 使用 ZooKeeper 的 multi-master Mesos 是以 `mesos://zk://host1:2181,host2:2181,host3:2181/mesos` 的形式表示。

## Client Mode（客户端模式）

在客户端模式下，Spark Mesos 框架直接在客户端机器上启动，并等待 driver 输出。

driver 需要在 `spark-env.sh` 中进行一些配置才能与 Mesos 进行交互：

1. 在 `spark-env.sh` 中设置一些环境变量：
 * `export MESOS_NATIVE_JAVA_LIBRARY=<path to libmesos.so>`. 这个路径通常是
   `<prefix>/lib/libmesos.so` 默认情况下前缀是 `/usr/local` 。请参阅上边的 Mesos 安装说明。在 Mac OS X 上，这个 library 叫做 `libmesos.dylib` 而不是 `libmesos.so` 。
 * `export SPARK_EXECUTOR_URI=<URL of spark-{{site.SPARK_VERSION}}.tar.gz uploaded above>`。
2. 还需要设置 `spark.executor.uri` 为 `<URL of spark-{{site.SPARK_VERSION}}.tar.gz>`。

现在，当针对集群启动一个 Spark 应用程序时，在创建 `SparkContext` 时传递一个 `mesos://` URL 作为 master 。例如：

{% highlight scala %}
val conf = new SparkConf()
  .setMaster("mesos://HOST:5050")
  .setAppName("My app")
  .set("spark.executor.uri", "<path to spark-{{site.SPARK_VERSION}}.tar.gz uploaded above>")
val sc = new SparkContext(conf)
{% endhighlight %}

(您还可以在 [conf/spark-defaults.conf](configuration.html#loading-default-configurations) 文件中使用 [`spark-submit`](submitting-applications.html) 并且配置 `spark.executor.uri` )

运行 shell 的时候，`spark.executor.uri` 参数从 `SPARK_EXECUTOR_URI` 继承，所以它不需要作为系统属性冗余地传入。

{% highlight bash %}
./bin/spark-shell --master mesos://host:5050
{% endhighlight %}

## Cluster mode（集群模式）

Spark on Mesos 还支持 cluster mode （集群模式），其中 driver 在集群中启动并且 client（客户端）可以在 Mesos Web UI 中找到 driver 的 results 。

要使用集群模式，你必须在您的集群中通过 `sbin/start-mesos-dispatcher.sh` 脚本启动 `MesosClusterDispatcher` ，传入 Mesos master URL （例如：mesos://host:5050）。这将启动 `MesosClusterDispatcher` 作为在主机上运行的守护程序。

如果您喜欢使用 Marathon 来运行 `MesosClusterDispatcher` ，您需要在 foreground （前台）运行 `MesosClusterDispatcher` （即 `bin/spark-class org.apache.spark.deploy.mesos.MesosClusterDispatcher`）。注意，`MesosClusterDispatcher` 尚不支持 HA 的多个实例。

`MesosClusterDispatcher` 还支持将 recovery state （恢复状态）写入 Zookeeper 。这将允许 `MesosClusterDispatcher` 能够在重新启动时恢复所有提交和运行的 containers （容器）。为了启用这个恢复模式，您可以在 spark-env 中通过配置 `spark.deploy.recoveryMode` 来设置 SPARK_DAEMON_JAVA_OPTS 和相关的 spark.deploy.zookeeper.* 配置。
有关这些配置的更多信息，请参阅配置 [doc](configurations.html#deploy) 。

从客户端，您可以提交一个作业到 Mesos 集群，通过执行 `spark-submit` 并指定 `MesosClusterDispatcher` 的 master URL （例如：mesos://dispatcher:7077）。您可以在 Spark cluster Web UI 查看 driver 的状态。

例如：
{% highlight bash %}
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  http://path/to/examples.jar \
  1000
{% endhighlight %}


请注意，传入到 spark-submit 的 jars 或者 python 文件应该是 Mesos slaves 可访问的 URIs ，因为 Spark driver 不会自动上传本地 jars 。

# Mesos 运行模式

Spark 可以以两种模式运行 Mesos ： "coarse-grained（粗粒度）"（默认） 和 "fine-grained（细粒度）"（不推荐）。

## Coarse-Grained（粗粒度）

在 "coarse-grained（粗粒度） 模式下，每个 Spark 执行器都作为单个 Mesos 任务运行。 Spark 执行器的大小是根据下面的配置变量确定的：

* Executor memory（执行器内存）: `spark.executor.memory`
* Executor cores（执行器核）: `spark.executor.cores`
* Number of executors（执行器数量）: `spark.cores.max`/`spark.executor.cores`

请参阅 [Spark 配置](configuration.html) 页面来了解细节和默认值。

当应用程序启动时，执行器就会高涨，直到达到 `spark.cores.max` 。如果您没有设置 `spark.cores.max` ，Spark 应用程序将会保留 Mesos 提供的所有的资源，因此我们当然会敦促您在任何类型的多租户集群上设置此变量，包括运行多个并发 Spark 应用程序的集群。

调度程序将会在提供的 Mesos 上启动执行器循环给它，但是没有 spread guarantees （传播保证），因为 Mesos 不提供这样的保证在提供流上。

在这个模式下 spark 执行器将遵守 port （端口）分配如果这些事由用户提供的。特别是如果用户在 Spark 配置中定义了 `spark.executor.port` 或者 `spark.blockManager.port` ，mesos 调度器将检查有效端口的可用 offers 包含端口号。如果没有这样的 range 可用，它会不启动任何任务。如果用户提供的端口号不受限制，临时端口像往常一样使用。如果用户定义了一个端口，这个端口实现意味着 one task per host （每个主机一个任务）。在未来网络中，isolation 将被支持。

粗粒度模式的好处是开销要低得多，但是在应用程序的整个持续时间内保留 Mesos 资源的代价。要配置您的作业以动态调整资源需求，请参阅 [动态分配](#dynamic-resource-allocation-with-mesos) 。

## Fine-Grained (deprecated)（细粒度，不推荐）

**注意:** Spark 2.0.0 中的细粒度模式已弃用。为了一些优点，请考虑使用 [动态分配](#dynamic-resource-allocation-with-mesos) 有关完整的解释，请参阅 [SPARK-11857](https://issues.apache.org/jira/browse/SPARK-11857) 。

在细粒度模式下，Spark 执行器中的每个 Spark 任务作为单独的 Mesos 任务运行。这允许 Spark 的多个实例（和其他框架）以非常细的粒度来共享 cores （内核），其中每个应用程序在其上升和下降时获得更多或更少的核。但是它在启动每个任务时增加额外的开销。这种模式可能不适合低延迟要求，如交互式查询或者提供 web 请求。

请注意，尽管细粒度的 Spark 任务在它们终止时将放弃内核，但是他们不会放弃内存，因为 JVM 不会将内存回馈给操作系统。执行器在空闲时也不会终止。

要以细粒度模式运行，请在您的 [SparkConf](configuration.html#spark-properties) 中设置 `spark.mesos.coarse` 属性为 false 。:

{% highlight scala %}
conf.set("spark.mesos.coarse", "false")
{% endhighlight %}

您还可以使用 `spark.mesos.constraints` 在 Mesos 资源提供上设置基于属性的约束。默认情况下，所有资源 offers 都将被接受。

{% highlight scala %}
conf.set("spark.mesos.constraints", "os:centos7;us-east-1:false")
{% endhighlight %}

例如，假设将 `spark.mesos.constraints` 设置为 `os:centos7;us-east-1:false` ，然后将检查资源 offers 以查看它们是否满足这两个约束，然后才会被接受以启动新的执行器。

# Mesos Docker 支持

Spark 可以通过在您的 [SparkConf](configuration.html#spark-properties) 中设置属性 `spark.mesos.executor.docker.image` 来使用 Mesos Docker 容器。

所使用的 Docker 图像必须有一个适合的版本的 Spark 已经是图像的一部分，也可以通过通常的方法让 Mesos 下载 Spark 。

需要 Mesos 的 0.20.1 版本或者更高版本。

请注意，默认情况下，如果 agent （代理程序中）的 Mesos 代理已经存在，则 Mesos agents 将不会 pull 图像。如果您使用 mutable image tags （可变图像标签）可以将 `spark.mesos.executor.docker.forcePullImage` 设置为 `true` ，以强制 agent 总是在运行执行器之前拉取 image 。Force pulling images （强制拉取图像）仅在 Mesos 0.22 版本及以上版本中可用。

# 集成 Hadoop 运行

您可以在现有的 Hadoop 集群集成运行 Spark 和 Mesos ，只需要在机器上启动他们作为分开的服务即可。要从 Spark 访问 Hadoop 数据，需要一个完整的 `hdfs://` URL （通常为 `hdfs://<namenode>:9000/path`），但是您可以在 Hadoop Namenode web UI 上找到正确的 URL 。

此外，还可以在 Mesos 上运行 Hadoop MapReduce，以便在两者之间实现更好的资源隔离和共享。 在这种情况下，Mesos 将作为统一的调度程序，将 Core 核心分配给 Hadoop 或 Spark，而不是通过每个节点上的 Linux 调度程序共享资源。 请参考 [Hadoop on Mesos](https://github.com/mesos/hadoop) 。

# 使用 Mesos 动态分配资源

Mesos 仅支持使用粗粒度模式的动态分配，这可以基于应用程序的统计信息调整执行器的数量。 有关一般信息，请参阅 [Dynamic Resource Allocation](job-scheduling.html#dynamic-resource-allocation) 。

要使用的外部 Shuffle 服务是 Mesos Shuffle 服务。 它在 Shuffle 服务之上提供 shuffle 数据清理功能，因为 Mesos 尚不支持通知另一个框架的终止。 要启动它，在所有从节点上运 `$SPARK_HOME/sbin/start-mesos-shuffle-service.sh` ，并将 `spark.shuffle.service.enabled` 设置为`true`。

这也可以通过 Marathon，使用唯一的主机约束和以下命令实现 : `bin/spark-class org.apache.spark.deploy.mesos.MesosExternalShuffleService`。

# 配置

有关 Spark 配置的信息，请参阅 [配置页面](configuration.html) 。以下配置特定于 Mesos 上的 Spark。

#### Spark 属性

<table class="table">
<tr><th>Property Name（属性名称）</th><th>Default（默认）</th><th>Meaning（含义）</th></tr>
<tr>
  <td><code>spark.mesos.coarse</code></td>
  <td>true</td>
  <td>
  如果设置为<code>true</code>，则以 “粗粒度” 共享模式在 Mesos 集群上运行，其中 Spark 在每台计算机上获取一个长期存在的 Mesos 任务。
如果设置为<code>false</code>，则以 “细粒度” 共享模式在 Mesos 集群上运行，其中每个 Spark 任务创建一个 Mesos 任务。
<a href="running-on-mesos.html#mesos-run-modes">'Mesos Run Modes'</a> 中的详细信息。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.extra.cores</code></td>
  <td><code>0</code></td>
  <td>
    设置执行程序公布的额外核心数。 这不会导致分配更多的内核。
它代替意味着执行器将“假装”它有更多的核心，以便驱动程序将发送更多的任务。
使用此来增加并行度。 此设置仅用于 Mesos 粗粒度模式。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.mesosExecutor.cores</code></td>
  <td><code>1.0</code></td>
  <td>
    （仅限细粒度模式）给每个 Mesos 执行器的内核数。 这不包括用于运行 Spark 任务的核心。
换句话说，即使没有运行 Spark 任务，每个 Mesos 执行器将占用这里配置的内核数。 该值可以是浮点数。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.executor.docker.image</code></td>
  <td>(none)</td>
  <td>
    设置 Spark 执行器将运行的 docker 映像的名称。所选映像必须安装 Spark，以及兼容版本的 Mesos 库。
Spark 在图像中的安装路径可以通过 <code>spark.mesos.executor.home</code> 来指定;
可以使用 <code>spark.executorEnv.MESOS_NATIVE_JAVA_LIBRARY</code> 指定 Mesos 库的安装路径。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.executor.docker.forcePullImage</code></td>
  <td>false</td>
  <td>
   强制 Mesos 代理拉取 <code> spark.mesos.executor.docker.image </code> 中指定的图像。
     默认情况下，Mesos 代理将不会拉取已经缓存的图像。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.executor.docker.parameters</code></td>
  <td>(none)</td>
  <td>
    在使用 docker 容器化器在 Mesos 上启动 Spark 执行器时，设置将被传递到<code> docker run </code>命令的自定义参数的列表。 此属性的格式是逗号分隔的列表
     键/值对。 例：
    <pre>key1=val1,key2=val2,key3=val3</pre>
  </td>
</tr>
<tr>
  <td><code>spark.mesos.executor.docker.volumes</code></td>
  <td>(none)</td>
  <td>
  设置要装入到 Docker 镜像中的卷列表，这是使用 <code>spark.mesos.executor.docker.image</code> 设置的。 此属性的格式是以逗号分隔的映射列表，后面的形式传递到 <code>docker run -v</code> 。 这是他们采取的形式 :
    <pre>[host_path:]container_path[:ro|:rw]</pre>
  </td>
</tr>
<tr>
  <td><code>spark.mesos.task.labels</code></td>
  <td>(none)</td>
  <td>
  设置 Mesos 标签以添加到每个任务。 标签是自由格式的键值对。
     键值对应以冒号分隔，并用逗号分隔多个。
    Ex. key:value,key2:value2.
  </td>
</tr>
<tr>
  <td><code>spark.mesos.executor.home</code></td>
  <td>driver side <code>SPARK_HOME</code></td>
  <td>
  在 Mesos 中的执行器上设置 Spark 安装目录。
默认情况下，执行器将只使用驱动程序的 Spark 主目录，它们可能不可见。
请注意，这只有当 Spark 二进制包没有通过 <code>spark.executor.uri</code> 指定时才是有意义的。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.executor.memoryOverhead</code></td>
  <td>executor memory * 0.10, with minimum of 384</td>
  <td>
    以每个执行程序分配的额外内存量（以 MB 为单位）。
默认情况下，开销将大于 <code>spark.executor.memory</code> 的 384 或 10%。
如果设置，最终开销将是此值。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.uris</code></td>
  <td>(none)</td>
  <td>
    当驱动程序或执行程序由 Mesos 启动时，要下载到沙箱的 URI 的逗号分隔列表。
这适用于粗粒度和细粒度模式。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.principal</code></td>
  <td>(none)</td>
  <td>
    设置 Spark 框架将用来与 Mesos 进行身份验证的主体。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.secret</code></td>
  <td>(none)</td>
  <td>
    设置 Spark 框架将用来与 Mesos 进行身份验证的机密。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.role</code></td>
  <td><code>*</code></td>
  <td>
    设置这个 Spark 框架对 Mesos 的作用。
角色在 Mesos 中用于预留和资源权重共享。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.constraints</code></td>
  <td>(none)</td>
  <td>
    基于属性的约束对 mesos 资源提供。 默认情况下，所有资源优惠都将被接受。
有关属性的更多信息，请参阅 <a href="http://mesos.apache.org/documentation/attributes-resources/">Mesos Attributes & Resources</a>
    <ul>
      <li>标量约束与 “小于等于” 语义匹配，即约束中的值必须小于或等于资源提议中的值。</li>
      <li>范围约束与 “包含” 语义匹配，即约束中的值必须在资源提议的值内。</li>
      <li>集合约束与语义的 “子集” 匹配，即约束中的值必须是资源提供的值的子集。</li>
      <li>文本约束与 “相等” 语义匹配，即约束中的值必须完全等于资源提议的值。</li>
      <li>如果没有作为约束的一部分存在的值，则将接受具有相应属性的任何报价（没有值检查）。</li>
    </ul>
  </td>
</tr>
<tr>
  <td><code>spark.mesos.containerizer</code></td>
  <td><code>docker</code></td>
  <td>
  这只影响 docker containers ，而且必须是 "docker" 或 "mesos"。 Mesos 支持两种类型 docker 的 containerizer："docker" containerizer，和首选 "mesos" containerizer。 在这里阅读更多：http://mesos.apache.org/documentation/latest/container-image/
  </td>
</tr>
<tr>
  <td><code>spark.mesos.driver.webui.url</code></td>
  <td><code>(none)</code></td>
  <td>
    设置 Spark Mesos 驱动程序 Web UI URL 以与框架交互。
如果取消设置，它将指向 Spark 的内部 Web UI。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.driverEnv.[EnvironmentVariableName]</code></td>
  <td><code>(none)</code></td>
  <td>
    这仅影响以群集模式提交的驱动程序。 添加由EnvironmentVariableName指定的环境变量驱动程序进程。 用户可以指定多个这些设置多个环境变量。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.dispatcher.webui.url</code></td>
  <td><code>(none)</code></td>
  <td>
    设置 Spark Mesos 分派器 Web UI URL 以与框架交互。
如果取消设置，它将指向 Spark 的内部 Web UI。
  </td>
  </tr>
<tr>
  <td><code>spark.mesos.dispatcher.driverDefault.[PropertyName]</code></td>
  <td><code>(none)</code></td>
  <td>
  设置驱动程序提供的默认属性通过 dispatcher。 例如，spark.mesos.dispatcher.driverProperty.spark.executor.memory=32g 导致在群集模式下提交的所有驱动程序的执行程序运行在 32g 容器中。
</td>
</tr>
<tr>
  <td><code>spark.mesos.dispatcher.historyServer.url</code></td>
  <td><code>(none)</code></td>
  <td>
    设置<a href="http://spark.apache.org/docs/latest/monitoring.html#viewing-after-the-fact">history
    server</a>。 然后，dispatcher 将链接每个驱动程序到其条目在历史服务器中。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.gpus.max</code></td>
  <td><code>0</code></td>
  <td>
  设置要为此作业获取的 GPU 资源的最大数量。 请注意，当没有找到 GPU 资源时，执行器仍然会启动因为这个配置只是一个上限，而不是保证数额。
  </td>
  </tr>
<tr>
  <td><code>spark.mesos.network.name</code></td>
  <td><code>(none)</code></td>
  <td>
  将 containers 附加到给定的命名网络。 如果这个作业是在集群模式下启动，同时在给定的命令中启动驱动程序网络。 查看 <a href="http://mesos.apache.org/documentation/latest/cni/"> Mesos CNI 文档</a>了解更多细节。
  </td>
</tr>
<tr>
  <td><code>spark.mesos.fetcherCache.enable</code></td>
  <td><code>false</code></td>
  <td>
  如果设置为 `true`，则所有 URI （例如：`spark.executor.uri`，
     `spark.mesos.uris`）将被<a
    HREF = "http://mesos.apache.org/documentation/latest/fetcher/"> Mesos
     Fetcher Cache</a>
  </td>
</tr>
</table>

# 故障排查和调试

在调试中可以看的地方：

- Mesos Master 的端口 : 5050
  - Slaves 应该出现在 Slaves 那一栏
  - Spark 应用应该出现在框架那一栏
  - 任务应该出现在在一个框架的详情
  - 检查失败任务的 sandbox 的 stdout 和 stderr
- Mesos 的日志
  - Master 和 Slave 的日志默认在 : `/var/log/mesos`  目录

常见的陷阱:

- Spark assembly 不可达/不可访问
  - Slave 必须可以从你给的 `http://`，`hdfs://` 或者 `s3n://` URL 地址下载的到 Spark 的二进制包
- 防火墙拦截通讯
  - 检查信息是否是连接失败
  - 临时禁用防火墙来调试，然后戳出适当的漏洞
