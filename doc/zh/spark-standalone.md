---
layout: global
title: Spark Standalone Mode
---

* This will become a table of contents (this text will be scraped).
{:toc}

Spark 除了运行在 Mesos 或者 YARN 上以外，Spark 还提供了一个简单的 standalone 部署模式。您可以手动启动 master 和 worker 来启动 standalone 集群，或者使用我们提供的 [launch scripts](#cluster-launch-scripts) 脚本。可以为了测试而在单个机器上运行这些进程。

# 安装 Spark Standalone 集群 

安装 Spark Standalone 集群，您只需要将编译好的版本部署在集群中的每个节点上。您可以获取 Spark 的每个版本的预编译版本或者自己编译 [build it yourself](building-spark.html).

# 手动启动一个集群

您可以启动一个 standalone master server 通过执行下面的代码：

    ./sbin/start-master.sh

一旦启动，master 将会为自己打印出一个 `spark://HOST:PORT` URL，您可以使用它来连接 workers，或者像传递 "master" 参数一样传递到 `SparkContext` 。您在 master 的web UI 上也会找到这个 URL ，默认情况下是 [http://localhost:8080](http://localhost:8080) 。

类似地，您可以启动一个或多个 workers 并且通过下面的代码连接到 master ：

    ./sbin/start-slave.sh <master-spark-URL>

在您启动一个 worker 之后，就可以通过 master 的 web UI ( 默认情况下是 [http://localhost:8080](http://localhost:8080))查看到了。您可以看到列出的新的 node （节点），以及其 CPU 的数量和数量（为操作系统留下了 1 GB 的空间）。

最后，下面的配置选项可以传递给 master 和 worker ：

<table class="table">
  <tr><th style="width:21%">Argument（参数）</th><th>Meaning（含义）</th></tr>
  <tr>
    <td><code>-h HOST</code>, <code>--host HOST</code></td>
    <td>监听的 Hostname</td>
  </tr>
  <tr>
    <td><code>-i HOST</code>, <code>--ip HOST</code></td>
    <td>监听的 Hostname （已弃用, 请使用 -h or --host）</td>
  </tr>
  <tr>
    <td><code>-p PORT</code>, <code>--port PORT</code></td>
    <td>监听的服务 Port （端口） （默认: master 是 7077 ， worker 是随机的）</td>
  </tr>
  <tr>
    <td><code>--webui-port PORT</code></td>
    <td>web UI 的端口（默认: master 是 8080 ， worker 是 8081）</td>
  </tr>
  <tr>
    <td><code>-c CORES</code>, <code>--cores CORES</code></td>
    <td>Spark 应用程序在机器上可以使用的全部的 CPU 核数（默认是全部可用的）；这个选项仅在 worker 上可用</td>
  </tr>
  <tr>
    <td><code>-m MEM</code>, <code>--memory MEM</code></td>
    <td>Spark 应用程序可以使用的内存数量，格式像 1000M 或者 2G（默认情况是您的机器内存数减去 1 GB）；这个选项仅在 worker 上可用</td>
  </tr>
  <tr>
    <td><code>-d DIR</code>, <code>--work-dir DIR</code></td>
    <td>用于 scratch space （暂存空间）和作业输出日志的目录（默认是：SPARK_HOME/work）；这个选项仅在 worker 上可用</td>
  </tr>
  <tr>
    <td><code>--properties-file FILE</code></td>
    <td>自定义的 Spark 配置文件加载目录（默认：conf/spark-defaults.conf）</td>
  </tr>
</table>


# 集群启动脚本

要使用启动脚本启动 Spark standalone 集群，你应该首先在 Spark 目录下创建一个叫做 conf/slaves 的文件，这个文件中必须包含所有你想要启动的 Spark workers 的机器的 hostname ，每个 hostname 占一行。
如果 conf/slaves 不存在，启动脚本默认启动单个机器（localhost），这对于测试是有效的。
注意， master 机器通过 ssh 访问所有的 worker 机器。默认情况下，ssh 是 parallel （并行）运行的并且需要配置无密码（使用一个私钥）的访问。
如果您没有设置无密码访问，您可以设置环境变量 SPARK_SSH_FOREGROUND 并且为每个 worker 提供一个密码。


一旦您创建了这个文件，您就可以启动或者停止您的集群使用下面的 shell 脚本，基于 Hadoop 的部署脚本，并在 `SPARK_HOME/sbin` 中可用：

- `sbin/start-master.sh` - 在执行的机器上启动一个 master 实例。
- `sbin/start-slaves.sh` - 在 `conf/slaves` 文件中指定的每个机器上启动一个 slave 实例。
- `sbin/start-slave.sh` - 在执行的机器上启动一个 slave 实例。
- `sbin/start-all.sh` - 启动一个 master 和上面说到的一定数量 slaves 。
- `sbin/stop-master.sh` - 停止通过 `sbin/start-master.sh` 脚本启动的 master。
- `sbin/stop-slaves.sh` - 停止在 `conf/slaves` 文件中指定的所有机器上的 slave 实例。
- `sbin/stop-all.sh` - 停止 master 和上边说到的 slaves 。

注意，这些脚本必须在您想要运行 Spark master 的机器上执行，而不是您本地的机器。

您可以通过在 `conf/spark-env.sh` 中设置环境变量来进一步配置集群。利用 `conf/spark-env.sh.template` 文件来创建这个文件，然后将它复制到所有的 worker 机器上使设置有效。下面的设置是可用的：

<table class="table">
  <tr><th style="width:21%">Environment Variable （环境变量）</th><th>Meaning（含义）</th></tr>
  <tr>
    <td><code>SPARK_MASTER_HOST</code></td>
    <td>绑定 master 到一个指定的 hostname 或者 IP 地址，例如一个 public hostname 或者 IP 。</td>
  </tr>
  <tr>
    <td><code>SPARK_MASTER_PORT</code></td>
    <td>在不同的端口上启动 master （默认：7077）</td>
  </tr>
  <tr>
    <td><code>SPARK_MASTER_WEBUI_PORT</code></td>
    <td>master 的 web UI 的端口（默认：8080）</td>
  </tr>
  <tr>
    <td><code>SPARK_MASTER_OPTS</code></td>
    <td>仅应用到 master 上的配置属性，格式是 "-Dx=y" （默认是：none）。查看下面的列表可能选项。</td>
  </tr>
  <tr>
    <td><code>SPARK_LOCAL_DIRS</code></td>
    <td>Spark 中 "scratch" space（暂存空间）的目录，包括 map 的输出文件和存储在磁盘上的 RDDs 。这必须在您的系统中的一个快速的，本地的磁盘上。这也可以是逗号分隔的不同磁盘上的多个目录的列表。
    </td>
  </tr>
  <tr>
    <td><code>SPARK_WORKER_CORES</code></td>
    <td>机器上 Spark 应用程序可以使用的全部的 cores（核）的数量。（默认：全部的核可用）</td>
  </tr>
  <tr>
    <td><code>SPARK_WORKER_MEMORY</code></td>
    <td>机器上 Spark 应用程序可以使用的全部的内存数量，例如 <code>1000m</code>， <code>2g</code>（默认：全部的内存减去 1 GB）；注意每个应用程序的<i>individual（独立）</i>内存是使用 <code>spark.executor.memory</code> 属性进行配置的。</td>
  </tr>
  <tr>
    <td><code>SPARK_WORKER_PORT</code></td>
    <td>在一个指定的 port （端口）上启动 Spark worker （默认： random（随机））</td>
  </tr>
  <tr>
    <td><code>SPARK_WORKER_WEBUI_PORT</code></td>
    <td>worker 的 web UI 的 Port （端口）（默认：8081）</td>
  </tr>
  <tr>
    <td><code>SPARK_WORKER_DIR</code></td>
    <td>运行应用程序的目录，这个目录中包含日志和暂存空间（default: SPARK_HOME/work）</td>
  </tr>
  <tr>
    <td><code>SPARK_WORKER_OPTS</code></td>
    <td>仅应用到 worker 的配置属性，格式是 "-Dx=y" （默认：none）。查看下面的列表的可能选项。</td>
  </tr>
  <tr>
    <td><code>SPARK_DAEMON_MEMORY</code></td>
    <td>分配给 Spark master 和 worker 守护进程的内存。（默认： 1g）</td>
  </tr>
  <tr>
    <td><code>SPARK_DAEMON_JAVA_OPTS</code></td>
    <td>Spark master 和 worker 守护进程的 JVM 选项，格式是 "-Dx=y" （默认：none）</td>
  </tr>
  <tr>
    <td><code>SPARK_PUBLIC_DNS</code></td>
    <td>Spark master 和 worker 的公开 DNS 名称。（默认：none）</td>
  </tr>
</table>

**注意：** 启动脚本现在还不支持 Windows 。要在 Windows 上运行一个 Spark 集群，需要手动启动 master 和 workers 。

SPARK_MASTER_OPTS 支持以下系统属性:

<table class="table">
<tr><th>Property Name（属性名称）</th><th>Default（默认）</th><th>Meaning（含义）</th></tr>
<tr>
  <td><code>spark.deploy.retainedApplications</code></td>
  <td>200</td>
  <td>
    展示的已完成的应用程序的最大数量。旧的应用程序将会从 UI 中被删除以满足限制。<br/>
  </td>
</tr>
<tr>
  <td><code>spark.deploy.retainedDrivers</code></td>
  <td>200</td>
  <td>
   展示已完成的 drivers 的最大数量。老的 driver 会从 UI 删除掉以满足限制。<br/>
  </td>
</tr>
<tr>
  <td><code>spark.deploy.spreadOut</code></td>
  <td>true</td>
  <td>
    这个选项控制 standalone 集群 manager 是应该跨界店 spread （传播）应用程序还是应该努力将应用程序整合到尽可能少的节点上。在 HDFS 中， Spreading 是数据本地化的更好的选择，但是对于计算密集型的负载，整合会更有效率。 <br/>
  </td>
</tr>
<tr>
  <td><code>spark.deploy.defaultCores</code></td>
  <td>(infinite)</td>
  <td>
    如果没有设置 <code>spark.cores.max</code> ，在 Spark 的 standalone 模式下默认分配给应用程序的 cores （核）数。如果没有设置，应用程序将总是获得所有的可用核，除非设置了 <code>spark.cores.max</code> 。在共享集群中设置较低的核数,可用于防止用户 grabbing （抓取）整个集群。<br/>
  </td>
</tr>
<tr>
  <td><code>spark.deploy.maxExecutorRetries</code></td>
  <td>10</td>
  <td>
    限制在 standalone 集群 manager 删除一个不正确地应用程序之前可能发生的 back-to-back 执行器失败的最大次数。如果一个应用程序有任何正在运行的执行器，则它永远不会被删除。如果一个应用程序经历过超过 <code>spark.deploy.maxExecutorRetries</code> 次的连续失败，没有执行器成功开始运行在这些失败之间，并且应用程序没有运行着的执行器，然后 standalone 集群 manager 将会移除这个应用程序并将它标记为失败。要禁用这个自动删除功能，设置<code>spark.deploy.maxExecutorRetries</code> 为 <code>-1</code> 。
    <br/>
  </td>
</tr>
<tr>
  <td><code>spark.worker.timeout</code></td>
  <td>60</td>
  <td>
    standalone 部署模式下 master 如果没有接收到心跳，认为一个 worker 丢失的间隔时间，秒数。
  </td>
</tr>
</table>

SPARK_WORKER_OPTS 支持以下的系统属性：

<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.worker.cleanup.enabled</code></td>
  <td>false</td>
  <td>
    激活周期性清空 worker / application 目录。注意，这只影响 standalone 模式，因为 YARN 工作方式不同。只有已停止的应用程序的目录会被清空。
  </td>
</tr>
<tr>
  <td><code>spark.worker.cleanup.interval</code></td>
  <td>1800 (30 minutes)</td>
  <td>
    在本地机器上，worker 控制清空老的应用程序的工作目录的时间间隔，以秒计数。
  </td>
</tr>
<tr>
  <td><code>spark.worker.cleanup.appDataTtl</code></td>
  <td>604800 (7 days, 7 * 24 * 3600)</td>
  <td>
    每个 worker 中应用程序工作目录的保留时间。这是一个 Live 时间，并且应该取决于您拥有的可用的磁盘空间量。应用程序的日志和 jars 都会被下载到应用程序的工作目录。随着时间的推移，这个工作目录会很快填满磁盘空间，特别是如果您经常运行作业。
  </td>
</tr>
<tr>
  <td><code>spark.worker.ui.compressedLogFileLengthCacheSize</code></td>
  <td>100</td>
  <td>
    对于压缩日志文件，只能通过未压缩文件来计算未压缩文件。
    Spark 缓存未压缩日志文件的未压缩文件大小。此属性控制缓存的大小。
  </td>
</tr>
</table>

# 提交应用程序到集群中

要在 Spark 集群中运行一个应用程序，只需要简单地将 master 的 `spark://IP:PORT` URL 传递到 [`SparkContext`
constructor](programming-guide.html#initializing-spark) 。

要针对集群运行交互式 Spark shell ，运行下面的命令：

    ./bin/spark-shell --master spark://IP:PORT

您还可以传递一个选项 `--total-executor-cores <numCores>` 来控制 spark-shell 在集群上使用的 cores （核）数。

# 启动 Spark 应用程序

[`spark-submit` 脚本](submitting-applications.html) 提供了最简单的方法将一个编译好的 Spark 应用程序提交到集群中。对于 standalone 集群，Spark 目前支持两种部署模式。在 `client` 模式下，driver 在与 client 提交应用程序相同的进程中启动。在 `cluster` 模式下，driver 是集群中的某个 Worker 中的进程中启动，并且 client 进程将会在完成提交应用程序的任务之后退出，而不需要等待应用程序完成再退出。

如果您的应用程序是通过 Spark 提交来启动的，则应用程序的 jar 将自动启动分发给所有的 worker 节点。对于您的应用程序依赖的其他的 jar ，您应该通过 `--jars` 标志使用逗号作为分隔符（例如 `--jars jar1,jar2`）来指定它们。
要控制应用程序的配置或执行环境，请参阅 [Spark Configuration](configuration.html) 。

另外，standalone `cluster` 模式支持自动重新启动应用程序如果它是以非零的退出码退出的。要使用此功能，您可以在启动您的应用程序的时候将 `--supervise` 标志传入 `spark-submit` 。然后，如果您想杀死一个重复失败的应用程序，您可以使用如下方式：

    ./bin/spark-class org.apache.spark.deploy.Client kill <master url> <driver ID>

您可以查看 driver ID 通过 standalone Master web UI 在 `http://<master url>:8080` 。

# Resource Scheduling（资源调度）

standalone 集群模式当前只支持一个简单的跨应用程序的 FIFO 调度。
然而，为了允许多个并发的用户，您可以控制每个应用程序能用的最大资源数。
默认情况下，它将获取集群中的 *all* cores （核），这只有在某一时刻只允许一个应用程序运行时才有意义。您可以通过 `spark.cores.max` 在 [SparkConf](configuration.html#spark-properties) 中设置 cores （核）的数量。例如：

{% highlight scala %}
val conf = new SparkConf()
  .setMaster(...)
  .setAppName(...)
  .set("spark.cores.max", "10")
val sc = new SparkContext(conf)
{% endhighlight %}

此外，您可以在集群的 master 进程中配置 `spark.deploy.defaultCores` 来修改默认为没有将 `spark.cores.max` 设置为小于无穷大的应用程序。
通过添加下面的命令到 `conf/spark-env.sh` 执行以上的操作：

{% highlight bash %}
export SPARK_MASTER_OPTS="-Dspark.deploy.defaultCores=<value>"
{% endhighlight %}

这在用户没有配置最大独立核数的共享的集群中是有用的。

# 监控和日志

Spark 的 standalone 模式提供了一个基于 web 的用户接口来监控集群。master 和每个 worker 都有它自己的显示集群和作业信息的 web UI 。 默认情况下，您可以通过 master 的 8080 端口来访问 web UI 。这个端口可以通过配置文件修改或者通过命令行选项修改。

此外，对于每个作业的详细日志输出也会写入到每个 slave 节点的工作目录中。（默认是 `SPARK_HOME/work`）。你会看到每个作业的两个文件，分别是 `stdout` 和 `stderr` ，其中所有输出都写入其控制台。


# 与 Hadoop 集成

您可以运行 Spark 集成到您现有的 Hadoop 集群，只需在同一台机器上将其作为单独的服务启动。要从 Spark 访问 Hadoop 的数据，只需要使用 hdfs:// URL (通常为 `hdfs://<namenode>:9000/path`, 但是您可以在您的 Hadoop Namenode 的 web UI 中找到正确的 URL 。)  或者，您可以为 Spark 设置一个单独的集群，并且仍然可以通过网络访问 HDFS ；这将比磁盘本地访问速度慢，但是如果您仍然在同一个局域网中运行（例如，您将 Hadoop 上的每个机架放置几台 Spark 机器），可能不会引起关注。


# 配置网络安全端口

Spark 对网络的需求比较高，并且一些环境对于使用严格的防火墙设置有严格的要求，请查看 [security page](security.html#configuring-ports-for-network-security) 。

# 高可用性

默认情况下， standalone 调度集群对于 Worker 的失败（对于 Spark 本身可以通过将其移动到其他 worker 来说是失败的工作而言）是有弹性的。但是，调度程序使用 Master 进行调度决策，并且（默认情况下）汇创建一个单点故障：如果 Master 崩溃，新的应用程序将不会被创建。为了规避这一点，我们有两个高可用性方案，详细说明如下。

## 使用 ZooKeeper 备用 Masters

**概述**

使用 ZooKeeper 提供的 leader election （领导选举）和一些 state storage （状态存储），在连接到同一 ZooKeeper 实例的集群中启动多个 Masters 。一个节点将被选举为 "leader" 并且其他节点将会维持备用模式。如果当前的 leader 宕掉了，另一个 Master 将会被选举，恢复老的 Master 的状态，并且恢复调度。整个恢复过程（从第一个 leader 宕掉开始）应该会使用 1 到 2 分钟。注意此延迟仅仅影响调度 _new_ 应用程序 -- 在 Master 故障切换期间已经运行的应用程序不受影响。

详细了解如何开始使用 ZooKeeper [这里](http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html) 。

**配置**

为了启用这个恢复模式，您可以在 spark-env 中设置 SPARK_DAEMON_JAVA_OPTS 通过配置 `spark.deploy.recoveryMode` 和相关的 spark.deploy.zookeeper.* 配置。
有关这些配置的更多信息，请参阅 [配置文档](configuration.html#deploy) 。

可能的陷阱：如果您在您的集群中有多个 Masters 但是没有正确地配置 Masters 使用 ZooKeeper ， Masters 将无法相互发现，并认为它们都是 leader 。这将不会形成一个健康的集群状态（因为所有的 Masters 将会独立调度）。

**细节**

在您设置了 ZooKeeper 集群之后，实现高可用性是很简单的。只需要在具有相同 ZooKeeper 配置（ZooKeeper URL 和 目录）的不同节点上启动多个 Master 进程。Masters 随时可以被添加和删除。

为了调度新的应用程序或者添加新的 Worker 到集群中，他们需要知道当前的 leader 的 IP 地址。这可以通过简单地传递一个您在一个单一的进程中传递的 Masters 的列表来完成。例如，您可以启动您的 SparkContext 指向 ``spark://host1:port1,host2:port2`` 。这将导致您的 SparkContext 尝试去注册两个 Masters -- 如果 ``host1`` 宕掉，这个配置仍然是正确地，因为我们将会发现新的 leader ``host2`` 。

在 "registering with a Master （使用 Master 注册）" 与正常操作之间有一个重要的区别。当启动的时候，一个应用程序或者 Worker 需要使用当前的 lead Master 找到并且注册。一旦它成功注册，它就是 "in the system（在系统中）"了（即存储在了 ZooKeeper 中）。如果发生故障切换，新的 leader 将会联系所有值钱已经注册的应用程序和 Workers 来通知他们领导层的变化，所以他们甚至不知道新的 Master 在启动时的存在。

由于这个属性，新的 Masters 可以在任何时间创建，唯一需要担心的是，_new_ 应用程序和 Workers 可以找到它注册，以防其成为 leader 。一旦注册了之后，您将被照顾。

## 用本地文件系统做单节点恢复

**概述**

ZooKeeper 是生产级别的高可用性的最佳方法，但是如果您只是想要重新启动 Master 服务器，如果 Master 宕掉，FILESYSTEM 模式将会关注它。当应用程序和 Workers 注册了之后，它们具有写入提供的目录的足够的状态，以便在重新启动 Master 进程的时候可以恢复它们。

**配置**

为了启用此恢复模式，你可以使用以下配置在 spark-env 中设置 SPARK_DAEMON_JAVA_OPTS ：

<table class="table">
  <tr><th style="width:21%">System property（系统属性）</th><th>Meaning（含义）</th></tr>
  <tr>
    <td><code>spark.deploy.recoveryMode</code></td>
    <td>设置为 FILESYSTEM 以启用单节点恢复模式（默认：NONE）</td>
  </tr>
  <tr>
    <td><code>spark.deploy.recoveryDirectory</code></td>
    <td>Spark 将存储恢复状态的目录，可以从 Master 的角度访问。</td>
  </tr>
</table>

**细节**

* 该解决方案可以与像 [monit](http://mmonit.com/monit/) 这样的过程 monitor/manager 一起使用，或者只是通过重新启动手动恢复。
* 尽管文件系统恢复似乎比完全没有任何恢复更好，但是对于某些特定的开发或者实验目的，此模式可能不太适合。特别是，通过 stop-master.sh 杀死 master 并不会清除其恢复状态，所以每当重新启动一个新的 Master 时，它将进入恢复模式。如果需要等待所有先前注册的 Worker/clients 超时，这可能会将启动时间增加 1 分钟。
* 虽然没有正式的支持，你也可以挂载 NFS 目录作为恢复目录。如果 original Master w安全地死亡，则您可以在不同的节点上启动 Master ，这将正确恢复所有以前注册的 Workers/applications （相当于 ZooKeeper 恢复）。然而，未来的应用程序必须能够找到新的 Master 才能注册。
